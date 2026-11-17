# PostgreSQL: porównanie krotek, które zjadło CPU bazy

Notatka z incydentu na produkcyjnym PostgreSQL (AWS RDS). Bez żadnego deployu CPU bazy skoczyło z ~3% do ~29%, a DBLoad z ~0,1 do ~1,1. Winne było jedno zapytanie, które od lat działało w poniżej 1 ms. Poprawka zmieniła trzy linijki SQL i nie zmieniła wyniku zapytania, tylko jego plan.

## Kontekst

Tabela `gps_events` to dziennik pozycji pojazdów: kilkadziesiąt GB, setki milionów wierszy, `id bigint` rośnie z czasem, wiele tenantów. Workery czytają ją kursorem keyset per tenant: „daj mi następne N wierszy tenanta po `id` = X, ale nie dalej niż znak wodny W”.

Ważne indeksy:

```sql
gps_events_pkey                      (id)
gps_events_tenant_id_cursor_idx      ("tenantId", id)
gps_events_tenant_vehicle_track_idx  ("tenantId", "vehicleId", "gpsTime" DESC, id DESC) INCLUDE (...)
```

Zapytanie strony kursora:

```sql
SELECT ... FROM gps_events g
WHERE ("tenantId", id) >  ($1::uuid, $2::bigint)
  AND ("tenantId", id) <= ($1::uuid, $3::bigint)
  AND "vehicleId" IS NOT NULL
  AND g."vehicleId" = ANY($5::uuid[])      -- filtr jednego z workerów: tylko śledzone pojazdy
ORDER BY g."tenantId", g.id
LIMIT $4
```

Zakres `($2, $3]` to zwykle kilkaset do kilku tysięcy id. Filtr pojazdów zawęża go do 1–2 pojazdów.

## Przebieg

- **04:24:05 UTC:** autovacuum robi `ANALYZE` na `gps_events`, bo zmieniło się wystarczająco dużo wierszy.
- **Od 04:25:** zapytanie strony dostaje nowy plan. Zamiast iść po `id` używa indeksu `("tenantId", "vehicleId", "gpsTime", id)` i czyta **całą historię** śledzonych pojazdów, odrzucając po drodze wszystko spoza zakresu id. Strona trwa ~5 s, wait event to `DataFileRead`, więc dane czytane są z dysku.
- Najpierw lecą timeouty po 15 s. Potem każde przejście workera trwa ~57 s (budżet 50 s plus ostatnia strona) i przetwarza 0–5 wierszy. Wcześniej przejście zajmowało 150–300 ms.
- **Pętla, która sama się podtrzymuje:** w trakcie wolnej strony przychodzą nowe pozycje, znak wodny ucieka do przodu, worker nigdy nie „dogania” i kręci się do końca budżetu. Przy następnym przejściu wszystko od nowa.

## Dlaczego planner się pomylił

### 1. Selektywność porównania krotek liczona jest tylko z pierwszej kolumny

W `src/backend/utils/adt/selfuncs.c` funkcja `rowcomparesel` traktuje `(a, b) > (x, y)` tak, jakby to było `a > x`:

> We estimate RowCompare selectivity by considering just the first (high order) columns, which makes it equivalent to an ordinary OpExpr.

Dla planera nasz warunek wygląda więc tak:

```
("tenantId", id) >  (T, lo)   ≈   "tenantId" >  T
("tenantId", id) <= (T, hi)   ≈   "tenantId" <= T
```

Kolumna `id` w ogóle nie wchodzi do oszacowania. Do tego `clauselist_selectivity` łączy pary `x > a AND x <= b` w jeden zakres tylko dla zwykłych porównań `kolumna op stała`. Dla krotek tego nie robi, więc dwie selektywności są po prostu mnożone, jakby były niezależne. Przy 3 tenantach o równej wielkości wychodzi `1/3 · 2/3 ≈ 22%` tabeli zamiast kilkuset wierszy.

### 2. Przy takim oszacowaniu indeks „po pojeździe” wygląda taniej

Planer „wie”, że zakres to ~22% tabeli, a filtr `vehicleId = ANY(...)` to 2 pojazdy z setek. Porównuje dwa plany:

- **po `id` (PK albo `(tenantId, id)`):** przejść „wielki” zakres i filtrować po pojeździe. W modelu planera to drogo.
- **po `(tenantId, vehicleId, …)`:** `tenantId` i `vehicleId` idą jako warunki indeksu, potem sortowanie kilku wierszy. W modelu planera to tanio.

W rzeczywistości zakres po `id` ma kilkaset wierszy, a historia dwóch pojazdów ma miliony. Warunek na `id` nie zawęża skanu indeksu, bo `id` jest dopiero czwartą kolumną klucza, za `gpsTime`. Postgres musi więc przejść wszystkie wpisy tych pojazdów.

### 3. Dlaczego akurat po `ANALYZE`

Plan był na granicy opłacalności od dawna, bo oszacowanie było złe od zawsze. Nowe statystyki (liczba wierszy, MCV i `n_distinct` dla `vehicleId`, histogramy) przesunęły koszty o tyle, że plan przeskoczył. Nikt niczego nie wdrożył. To typowy **plan flip**: przyczyna leży w kodzie, ale wyzwala ją zmiana statystyk.

## Poprawka

```diff
 SELECT … FROM gps_events g
-WHERE ("tenantId", id) >  ($1::uuid, $2::bigint)
-  AND ("tenantId", id) <= ($1::uuid, $3::bigint)
+WHERE g."tenantId" = $1::uuid
+  AND g.id >  $2::bigint
+  AND g.id <= $3::bigint
   AND "vehicleId" IS NOT NULL
   [AND (pageFilter)]
 ORDER BY g."tenantId", g.id
 LIMIT $4
```

**Równoważność.** Porównanie krotek jest leksykograficzne:

```
(t, id) > (T, lo)   ⇔   t > T  OR (t = T AND id > lo)
(t, id) <= (T, hi)  ⇔   t < T  OR (t = T AND id <= hi)
```

Koniunkcja obu wymaga, żeby `t > T` albo `t = T` z pierwszego warunku i jednocześnie `t < T` albo `t = T` z drugiego. Spełnić oba naraz można tylko przy `t = T`, więc wynik to `t = T AND lo < id <= hi`. Te same wiersze, ta sama kolejność.

**Efekt na produkcji**, przy tych samych parametrach: `Index Scan using gps_events_pkey`, 129 buforów, **0,2 ms** zamiast ~5 s. CPU wróciło do normy.

Zwykłe `id > lo AND id <= hi` planer rozpoznaje jako zakres i szacuje z histogramu `id`. Zna więc prawdziwą liczbę wierszy i wybiera skan po `id`.

## Reprodukcja (lokalnie, ~30 s)

```sql
CREATE TABLE ev (id bigint PRIMARY KEY, tenant int NOT NULL, vehicle int NOT NULL,
                 ts timestamptz NOT NULL, payload text);
INSERT INTO ev
SELECT g, 1 + (g % 3), 1 + (g % 500), now() - interval '1 second' * (2000000 - g), repeat('x', 50)
FROM generate_series(1, 2000000) g;
CREATE INDEX ev_tenant_id_idx         ON ev (tenant, id);
CREATE INDEX ev_tenant_vehicle_ts_idx ON ev (tenant, vehicle, ts DESC, id DESC);
ANALYZE ev;

-- A: krotki — zwróć uwagę na rows=
EXPLAIN SELECT * FROM ev
WHERE (tenant, id) > (2, 1999000) AND (tenant, id) <= (2, 1999600)
ORDER BY tenant, id LIMIT 5000;

-- B: proste warunki
EXPLAIN SELECT * FROM ev
WHERE tenant = 2 AND id > 1999000 AND id <= 1999600
ORDER BY tenant, id LIMIT 5000;

-- C i D: to samo z filtrem pojazdów, z ANALYZE, BUFFERS
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM ev
WHERE (tenant, id) > (2, 1999000) AND (tenant, id) <= (2, 1999600) AND vehicle = ANY('{2,5}')
ORDER BY tenant, id LIMIT 5000;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM ev
WHERE tenant = 2 AND id > 1999000 AND id <= 1999600 AND vehicle = ANY('{2,5}')
ORDER BY tenant, id LIMIT 5000;
```

Wynik na PostgreSQL 16:

| Wariant | Szacunek `rows` | Plan | Bufory | Czas |
|---|---|---|---|---|
| A: krotki | **445 506** (prawda: 200) | `ev_tenant_id_idx` | – | – |
| B: proste | 225 | `ev_pkey` | – | – |
| C: krotki + pojazdy | 1 770 | Bitmap na `ev_tenant_vehicle_ts_idx` + Sort | 6 575 | 25,6 ms |
| D: proste + pojazdy | 1 | `ev_pkey` + Filter | **14** | **0,07 ms** |

`445 506 ≈ 2 000 000 · ⅓ · ⅔`, czyli dokładnie mnożenie selektywności samej pierwszej kolumny. Na 2 mln wierszy różnica to 25 ms wobec 0,07 ms. Na produkcji, gdzie historia pojazdu ma miliony wpisów i nie mieści się w cache, wychodziło ~5 s.

## Jak to diagnozować

1. **Metryki bazy:** CPU i DBLoad (Performance Insights) pokazują, *kiedy* się zaczęło. Wait event `DataFileRead` mówi, że zapytanie czyta z dysku dużo więcej, niż powinno.
2. **Co zbiegło się w czasie:** deploy? Jeśli nie, sprawdź statystyki:
   ```sql
   SELECT relname, last_autoanalyze, last_autovacuum, n_mod_since_analyze
   FROM pg_stat_user_tables WHERE relname = 'gps_events';
   ```
3. **Które zapytanie:** `pg_stat_activity` (zapytania trwające sekundy, ich `wait_event`) oraz `pg_stat_statements` (wzrost `total_exec_time`/`shared_blks_read`).
4. **Plan z prawdziwymi parametrami:** `EXPLAIN (ANALYZE, BUFFERS)`. Najważniejsza jest różnica między szacowanym a faktycznym `rows`. Jeśli różnią się o kilka rzędów wielkości, to tu szukaj przyczyny.
5. **Czasy jobów aplikacji:** u nas tabela z czasem każdego przejścia workera pokazała minutę przeskoku z dokładnością do sekund.

## Reguły kciuka

- **Krotki `(a, b) > (x, y)` są dobre do paginacji keyset, gdy `a` naprawdę się zmienia,** np. `ORDER BY created_at, id`. Wtedy są czytelne i dają warunek indeksu na `(a, b)`. Gdy `a` jest ustalone równością, rozpisz to na `a = x AND b > y`. Planer szacuje proste porównania dużo lepiej.
- **Pamiętaj, że planer szacuje krotkę z pierwszej kolumny.** Statystyki rozszerzone (`CREATE STATISTICS`) tu nie pomogą, bo `rowcomparesel` z nich nie korzysta.
- **Plan może się zmienić bez deployu.** Szybkie zapytania na gorącej ścieżce trzeba monitorować pod kątem czasu, nie tylko błędów. Pomaga alarm na CPU bazy, a jeszcze lepiej na czas jobu.
- **Budżet czasowy pętli z ruchomym celem** („czytaj aż dogonisz znak wodny”) zamienia wolne zapytanie w stałe obciążenie. Warto logować, ile wierszy przypada na sekundę przejścia. Nagły spadek do zera przy pełnym budżecie to sygnał alarmowy.
- **Przy poprawce dowiedź równoważności logicznej** i porównaj plan przed i po na tych samych parametrach, na produkcyjnych statystykach. Lokalne dane często mają inne statystyki i dają inny plan.
- **Przeszukaj kod pod kątem tego samego wzorca.** Inne zapytanie z tym samym `("tenantId", id) > (...)` może przeskoczyć przy następnym `ANALYZE`.

## Źródła

- PostgreSQL: [Row Constructor Comparison](https://www.postgresql.org/docs/current/functions-comparisons.html#ROW-WISE-COMPARISON)
- PostgreSQL: [How the Planner Uses Statistics](https://www.postgresql.org/docs/current/planner-stats-details.html)
- Kod: `rowcomparesel` w `src/backend/utils/adt/selfuncs.c`, łączenie zakresów w `clauselist_selectivity` (`src/backend/optimizer/path/clausesel.c`)
