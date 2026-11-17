# PostgreSQL: jak jedno zapytanie zjadło CPU bazy, choć nikt nic nie wdrożył

Notatka z incydentu na produkcyjnym PostgreSQL (AWS RDS). Pewnego ranka CPU bazy wzrosło z ~3% do ~29%, a jeden z workerów aplikacji zaczął działać kilkaset razy wolniej. Tego dnia nie było żadnego deployu. Przyczyną było zapytanie, które od dawna wykonywało się w ułamku milisekundy, aż PostgreSQL po odświeżeniu statystyk sam zdecydował się wykonywać je inaczej.

Żeby zrozumieć, co się stało, trzeba znać kilka mechanizmów PostgreSQL. Najpierw opisuję je po kolei (części 1–6), potem przebieg incydentu (część 7), poprawkę (część 8), lokalną reprodukcję (część 9) i wnioski (części 10–11).

---

## 1. Tabela i indeksy w tej historii

Tabela `gps_events` przechowuje pozycje GPS wysyłane przez urządzenia w pojazdach. Ma kilkadziesiąt GB i setki milionów wierszy. Najważniejsze kolumny:

| Kolumna | Znaczenie |
|---|---|
| `id` | `bigint`, kolejny numer wiersza. Nowsze wiersze mają większe `id`. |
| `tenantId` | Klient (firma), do którego należy pojazd. Klientów jest kilku. |
| `vehicleId` | Pojazd. Pojazdów są setki, każdy ma miliony wierszy historii. |
| `gpsTime` | Czas pomiaru pozycji. |
| `receivedAt` | Czas, w którym serwer odebrał wiersz. |

Na tabeli są m.in. trzy indeksy:

```sql
gps_events_pkey                      (id)
gps_events_tenant_id_cursor_idx      ("tenantId", id)
gps_events_tenant_vehicle_track_idx  ("tenantId", "vehicleId", "gpsTime" DESC, id DESC)
```

## 2. Indeks B-tree z kilkoma kolumnami: kolejność ma znaczenie

Indeks B-tree to posortowana lista wpisów (klucz → wskaźnik na wiersz w tabeli). Postgres potrafi bardzo szybko znaleźć w niej miejsce startu, a potem czyta kolejne wpisy po kolei.

W indeksie wielokolumnowym wpisy są posortowane najpierw po pierwszej kolumnie, w jej obrębie po drugiej, i tak dalej, jak słowa w słowniku. Indeks `("tenantId", "vehicleId", "gpsTime" DESC, id DESC)` wygląda więc tak:

```
tenant A, pojazd 1, 2026-11-17 10:00, id 9001
tenant A, pojazd 1, 2026-11-17 09:59, id 8995
tenant A, pojazd 1, ...                      ← cała historia pojazdu 1, od najnowszych
tenant A, pojazd 1, 2024-01-03 06:00, id 17
tenant A, pojazd 2, 2026-11-17 10:00, id 9003
...
```

Skutek: indeks dobrze zawęża wyszukiwanie po kolumnach z **początku** klucza. Warunek `tenantId = A AND vehicleId = 1` wskazuje dokładnie jeden spójny kawałek indeksu. Warunek na `id` (czwartą kolumnę) w ogóle nie zawęża tego kawałka, bo wpisy o `id` z zadanego przedziału są rozrzucone po całej historii pojazdu, między wpisami z innymi `gpsTime`. Żeby znaleźć wiersze pojazdu z `id` między 1000 a 1600, Postgres musi przejść **całą historię pojazdu** i każdy wpis sprawdzić osobno.

W `EXPLAIN` widać to jako dwie różne linijki:

- `Index Cond:` to warunki, które **zawężają** odczytywany fragment indeksu. Są tanie.
- `Filter:` to warunki sprawdzane **dla każdego przeczytanego wiersza**. Nie zmniejszają ilości czytanych danych. `Rows Removed by Filter` mówi, ile pracy poszło na marne.

## 3. Planer i model kosztów

To samo zapytanie SQL da się wykonać na wiele sposobów: przez różne indeksy, z sortowaniem albo bez. Wybiera **planer** (optymalizator). Działa tak:

1. Generuje możliwe plany.
2. Dla każdego szacuje **koszt**, czyli umowną liczbę reprezentującą odczyty stron z dysku i pracę CPU.
3. Wybiera plan o najniższym szacowanym koszcie.

Koszt zależy przede wszystkim od tego, **ile wierszy według planera spełni każdy warunek**. Tę liczbę planer nie zna, tylko ją **szacuje**. Jeśli oszacowanie jest złe, planer liczy koszty dobrze, ale na złych danych, i wybiera zły plan.

`EXPLAIN` pokazuje plan i szacunki (`rows=`). `EXPLAIN (ANALYZE, BUFFERS)` dodatkowo wykonuje zapytanie i pokazuje rzeczywiste liczby (`actual ... rows=`) oraz liczbę przeczytanych stron 8 KB (`Buffers: shared hit=... read=...`). Rozjazd szacowanego i rzeczywistego `rows` o kilka rzędów wielkości prawie zawsze jest źródłem problemu.

### LIMIT w modelu kosztów

Zapytanie z `LIMIT 5000` można przerwać po znalezieniu 5000 wierszy. Planer zakłada, że pasujące wiersze są **równomiernie rozrzucone**. Jeśli w danym kawałku indeksu spodziewa się 1% pasujących wierszy, liczy, że do zebrania 5000 musi przeczytać ~500 000 wpisów. Jeśli spodziewa się, że pasujących wierszy jest mniej niż `LIMIT`, zakłada, że przeczyta **cały** kawałek do końca.

## 4. Statystyki: skąd planer bierze oszacowania

Planer nie liczy wierszy w trakcie planowania. Korzysta ze **statystyk** zapisanych w katalogu systemowym (czytelny widok to `pg_stats`). Dla każdej kolumny są to m.in.:

- **MCV** (*most common values*): lista najczęstszych wartości i ich udziałów, np. „tenant A to 45% wierszy”.
- **histogram**: granice przedziałów, z których każdy zawiera tyle samo wierszy. Dzięki niemu planer szacuje warunki zakresowe typu `id > X AND id <= Y`.
- **n_distinct**: ile jest różnych wartości.
- oraz łączna liczba wierszy i stron tabeli.

Z tego planer liczy **selektywność** warunku, czyli jaki ułamek wierszy go spełni. Przykład: `tenantId = A` → 0,45 z listy MCV, a `id > 1000 AND id <= 1600` → kilkaset wierszy z histogramu.

Kilka warunków połączonych `AND` planer zwykle **mnoży**, zakładając, że są niezależne: selektywność `a AND b` = selektywność(a) × selektywność(b). Jest jeden ważny wyjątek: dwa warunki na tej samej kolumnie w postaci `kolumna > X` i `kolumna <= Y` planer rozpoznaje jako **przedział** i szacuje razem z histogramu. Tak jest dokładniej, bo pomnożenie „połowa wierszy ma `id > X`” przez „połowa ma `id <= Y`” dałoby bzdurne 25%.

### Kiedy statystyki się zmieniają: ANALYZE i autoanalyze

Statystyki zbiera polecenie `ANALYZE`. Czyta losową próbkę wierszy (domyślnie 30 000) i na jej podstawie buduje MCV, histogramy itd. Uruchamia je ręcznie człowiek albo automatycznie proces **autovacuum** (wtedy nazywa się to *autoanalyze*), gdy od poprzedniej analizy zmieni się dość dużo wierszy. Próg to `autovacuum_analyze_threshold + autovacuum_analyze_scale_factor × liczba_wierszy`; domyślnie 50 + 10% tabeli, a dostawcy chmurowi i administratorzy często to zmieniają (`SHOW autovacuum_analyze_scale_factor`).

Kiedy było ostatnie analyze, sprawdzisz tak:

```sql
SELECT relname, last_analyze, last_autoanalyze, n_mod_since_analyze
FROM pg_stat_user_tables WHERE relname = 'gps_events';
```

Co z tego wynika: **na tabeli, do której ciągle coś dochodzi, statystyki zmieniają się same, regularnie i bez udziału człowieka**, a każda zmiana może przestawić wybór planera.

## 5. Porównanie krotek i jak planer je szacuje

PostgreSQL pozwala porównywać kilka wartości naraz:

```sql
WHERE ("tenantId", id) > ('A', 1000)
```

To porównanie **leksykograficzne**, jak przy sortowaniu słów: najpierw porównywana jest pierwsza kolumna, a druga dopiero wtedy, gdy pierwsze są równe. Zapis powyżej znaczy dokładnie:

```sql
WHERE "tenantId" > 'A'
   OR ("tenantId" = 'A' AND id > 1000)
```

Takie warunki są wygodne przy **paginacji keyset**, czyli stronicowaniu „daj następne N wierszy po ostatnim widzianym kluczu” zamiast `OFFSET`. Dają też indeksowi `("tenantId", id)` dokładny punkt startu.

Haczyk jest w szacowaniu. Funkcja `rowcomparesel` (plik `src/backend/utils/adt/selfuncs.c`) liczy selektywność porównania krotek **tylko z pierwszej kolumny**. Komentarz w kodzie mówi wprost:

> We estimate RowCompare selectivity by considering just the first (high order) columns, which makes it equivalent to an ordinary OpExpr.

Planer widzi zatem:

```
("tenantId", id) >  ('A', 1000)   →  szacuje jak  "tenantId" >  'A'
("tenantId", id) <= ('A', 1600)   →  szacuje jak  "tenantId" <= 'A'
```

To powoduje dwa błędy naraz:

1. **Kolumna `id` w ogóle nie wpływa na oszacowanie.** Planer nie wie, że chodzi o wąski przedział 600 numerów.
2. **Wyjątek „przedziałowy” z części 4 nie działa.** Działa on tylko dla prostych warunków `kolumna op wartość`, a nie dla krotek. Planer mnoży więc dwie selektywności jak niezależne.

Przy trzech równych klientach wychodzi: udział „`tenantId` > A” ≈ 1/3, udział „`tenantId` <= A” ≈ 2/3, razem 1/3 × 2/3 ≈ **22% tabeli**. W rzeczywistości warunek wybiera kilkaset wierszy.

Statystyki rozszerzone (`CREATE STATISTICS`) tu nie pomogą, bo `rowcomparesel` z nich nie korzysta.

## 6. Dysk, cache i czas

Postgres trzyma ostatnio używane strony tabel i indeksów w pamięci (`shared_buffers`, do tego cache systemu operacyjnego). Strona z pamięci to `shared hit`, strona z dysku to `read`. Gdy zapytanie czyta dane, których nikt dawno nie używał (np. historia pojazdu sprzed dwóch lat), większość stron trzeba ściągnąć z dysku. W statystykach aktywności (`pg_stat_activity.wait_event`, AWS Performance Insights) widać wtedy zdarzenie oczekiwania **`DataFileRead`**: proces czeka na odczyt pliku danych. Duży udział `DataFileRead` w obciążeniu bazy oznacza, że jakieś zapytanie czyta znacznie więcej danych, niż powinno.

---

## 7. Co się stało

### Zapytanie

Kilka workerów aplikacji czyta `gps_events` kursorem: każdy pamięta per klient, do którego `id` doszedł, i co chwilę pyta o nowe wiersze. Jedna strona to:

```sql
SELECT ... FROM gps_events g
WHERE ("tenantId", id) >  ($1, $2)       -- $2 = id, na którym worker skończył
  AND ("tenantId", id) <= ($1, $3)       -- $3 = watermark (patrz niżej)
  AND "vehicleId" IS NOT NULL
  AND g."vehicleId" = ANY($5)            -- tylko pojazdy, które ten worker śledzi (1–2)
ORDER BY g."tenantId", g.id
LIMIT $4                                  -- 5000
```

Zakres `($2, $3]` ma zwykle kilkaset do kilku tysięcy `id`.

### Watermark: do którego `id` wolno czytać

Watermark (górna granica) to największe `id`, do którego worker może bezpiecznie dojść. Nie jest to po prostu `max(id)`. Wiersze zapisuje kilka procesów naraz, a `id` jest przydzielane w momencie `INSERT`, nie przy `COMMIT`. Może się więc zdarzyć, że wiersz 106 jest już zatwierdzony i widoczny, a 105 wciąż siedzi w otwartej transakcji. Gdyby worker przeskoczył na 106, wiersz 105 pojawiłby się później **za** kursorem i nikt by go nie przeczytał.

Dlatego watermark to „id tuż przed najstarszym wierszem odebranym w ostatnich 30 sekundach”. Wiersze starsze niż 30 s są na pewno zatwierdzone, więc wszystko poniżej tej granicy jest kompletne. Watermark przesuwa się do przodu z każdą nową porcją danych od urządzeń.

Jedno przejście workera to pętla z limitem czasu 50 s:

```
dopóki nie minęło 50 s:
    W = policz watermark                 # przed każdą stroną od nowa
    jeśli kursor >= W: koniec przejścia  # dogoniliśmy nowe dane
    przeczytaj stronę (kursor, W]
    kursor = W
```

### Normalnie

Planer wybierał indeks `gps_events_tenant_id_cursor_idx` albo klucz główny i szedł po kolei po `id` z przedziału, odrzucając wiersze innych pojazdów. Kilkaset wpisów, poniżej 1 ms. Następne obliczenie watermarku dawało tę samą wartość, więc przejście kończyło się po 150–300 ms.

### Po autoanalyze o 04:24

Autovacuum zebrał nowe statystyki `gps_events`. Przy następnym planowaniu planer policzył koszty tak:

- **Plan „po `id`”:** przedział według oszacowania z części 5 to ~22% tabeli, czyli dziesiątki milionów wierszy. Śledzone pojazdy to 2 z setek, więc pasujących wierszy planer spodziewa się mało, mniej niż `LIMIT`. Według części 3 oznacza to, że trzeba przejść cały przedział. **Bardzo drogo.**
- **Plan „po pojeździe”** (`gps_events_tenant_vehicle_track_idx`): `tenantId` i `vehicleId` idą jako `Index Cond` i wskazują historię dwóch pojazdów. Warunek na `id` sprawdzany jest per wpis, a na końcu jest sortowanie wyniku po `id`. **W modelu tanio.**

Planer wybrał drugi plan. Rzeczywistość była odwrotna: przedział po `id` miał kilkaset wierszy, a historia dwóch pojazdów miliony wpisów, w dużej części nieobecnych w cache. Każda strona trwała ~5 s, prawie cały czas w `DataFileRead`.

Dlaczego dopiero teraz? Oszacowanie z części 5 było złe od zawsze. Koszty obu planów były jednak na tyle blisko, że wygrywał dobry. Nowe statystyki (liczba wierszy, częstości pojazdów, histogramy) przesunęły wagę na drugą stronę. Takie zjawisko nazywa się **plan flip**: zapytanie i dane są te same, ale zmiana statystyk zmienia plan.

### Dlaczego to dało stałe obciążenie, a nie jedno wolne zapytanie

Wróćmy do pętli z watermarkiem. Strona trwa teraz 5 s. W tym czasie urządzenia całego klienta (nie tylko śledzone pojazdy) dosyłają nowe pozycje, więc nowo policzony watermark jest dalej niż kursor. Warunek „dogoniliśmy” nie zachodzi, jest kolejna 5-sekundowa strona, w tym czasie przychodzą nowe dane i tak w kółko, aż do limitu 50 s. Każde przejście trwało ~57 s (limit plus ostatnia rozpoczęta strona) i przetwarzało 0–5 wierszy. Kolejne przejście startowało od razu. Baza bez przerwy czytała z dysku historię tych samych pojazdów: stąd wzrost CPU z ~3% do ~29% i obciążenia (DBLoad) z ~0,1 do ~1,1.

## 8. Poprawka

```diff
 SELECT … FROM gps_events g
-WHERE ("tenantId", id) >  ($1, $2)
-  AND ("tenantId", id) <= ($1, $3)
+WHERE g."tenantId" = $1
+  AND g.id >  $2
+  AND g.id <= $3
   AND "vehicleId" IS NOT NULL
   AND g."vehicleId" = ANY($5)
 ORDER BY g."tenantId", g.id
 LIMIT $4
```

**Wynik jest dokładnie ten sam.** Rozpiszmy oba warunki krotkowe (części 5), oznaczając `t` = `tenantId`, a `T` = szukany klient:

```
(t, id) >  (T, lo)    ⇔   t > T   lub  (t = T  i  id > lo)
(t, id) <= (T, hi)    ⇔   t < T   lub  (t = T  i  id <= hi)
```

Oba muszą być prawdziwe naraz. Pierwszy wymaga `t ≥ T`, drugi `t ≤ T`, więc jedyna możliwość to `t = T`. Wtedy z pierwszego zostaje `id > lo`, a z drugiego `id <= hi`. Razem: `t = T AND lo < id <= hi`, czyli dokładnie nowy warunek.

**Dlaczego to pomaga.** `id > lo AND id <= hi` to dwa proste warunki na tej samej kolumnie, więc planer rozpoznaje przedział i szacuje go z histogramu `id` (część 4). Zna więc prawdziwą liczbę kilkuset wierszy. Plan „po `id`” jest wtedy w jego modelu tani i wygrywa zgodnie z rzeczywistością.

**Efekt na produkcji**, przy tych samych parametrach: `Index Scan using gps_events_pkey`, 129 stron, **0,2 ms** zamiast ~5 s. CPU wróciło do normy po wdrożeniu.

## 9. Reprodukcja na własnym komputerze

Wystarczy dowolny PostgreSQL (sprawdzone na 16). Tabela z 2 mln wierszy, 3 klientami i 500 pojazdami, tworzenie trwa ~30 s:

```sql
CREATE TABLE ev (id bigint PRIMARY KEY, tenant int NOT NULL, vehicle int NOT NULL,
                 ts timestamptz NOT NULL, payload text);
INSERT INTO ev
SELECT g, 1 + (g % 3), 1 + (g % 500), now() - interval '1 second' * (2000000 - g), repeat('x', 50)
FROM generate_series(1, 2000000) g;
CREATE INDEX ev_tenant_id_idx         ON ev (tenant, id);
CREATE INDEX ev_tenant_vehicle_ts_idx ON ev (tenant, vehicle, ts DESC, id DESC);
ANALYZE ev;

-- A: krotki — popatrz na rows= (ile wierszy według planera)
EXPLAIN SELECT * FROM ev
WHERE (tenant, id) > (2, 1999000) AND (tenant, id) <= (2, 1999600)
ORDER BY tenant, id LIMIT 5000;

-- B: proste warunki — to samo zapytanie logicznie
EXPLAIN SELECT * FROM ev
WHERE tenant = 2 AND id > 1999000 AND id <= 1999600
ORDER BY tenant, id LIMIT 5000;

-- C: krotki + filtr dwóch pojazdów, z wykonaniem
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM ev
WHERE (tenant, id) > (2, 1999000) AND (tenant, id) <= (2, 1999600) AND vehicle = ANY('{2,5}')
ORDER BY tenant, id LIMIT 5000;

-- D: proste warunki + filtr dwóch pojazdów, z wykonaniem
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM ev
WHERE tenant = 2 AND id > 1999000 AND id <= 1999600 AND vehicle = ANY('{2,5}')
ORDER BY tenant, id LIMIT 5000;
```

Co wychodzi:

| Wariant | Planer myśli, że wierszy jest | Wybrany indeks | Przeczytane strony | Czas |
|---|---|---|---|---|
| A: krotki | **445 506** (naprawdę ~200) | `ev_tenant_id_idx` | – | – |
| B: proste | 225 | `ev_pkey` | – | – |
| C: krotki + pojazdy | 1 770 | `ev_tenant_vehicle_ts_idx` + sortowanie | 6 575 | 25,6 ms |
| D: proste + pojazdy | 1 | `ev_pkey` + Filter | **14** | **0,07 ms** |

Jak to czytać:

- **A:** `445 506 ≈ 2 000 000 × ⅓ × ⅔`. To dokładnie mnożenie selektywności samej kolumny `tenant` z części 5. Kolumna `id` nie miała żadnego wpływu.
- **B:** proste warunki, więc planer rozpoznał przedział i oszacował 225 wierszy, prawie trafnie.
- **C:** planer uznał przedział za ogromny i poszedł przez indeks pojazdów. Przeczytał 6 575 stron (całą historię dwóch pojazdów), żeby znaleźć 2 wiersze.
- **D:** szedł po `id` przez 600 wierszy przedziału i 598 odrzucił filtrem (`Rows Removed by Filter: 598`). To tylko 14 stron.

Na 2 mln wierszy w pamięci różnica to 25 ms wobec 0,07 ms. Na produkcji, gdzie historia pojazdu to miliony wpisów czytanych z dysku, było to ~5 s na stronę.

## 10. Jak szukać takiego problemu

1. **Kiedy się zaczęło:** wykres CPU i obciążenia bazy (w AWS: CloudWatch `CPUUtilization`, Performance Insights `DBLoad`) daje minutę startu, a podział na wait events pokazuje, czy to `DataFileRead` (za dużo czytania).
2. **Co się wtedy stało:** deploy? Jeśli nie było, sprawdź `last_autoanalyze` w `pg_stat_user_tables` (część 4). Zbieżność co do minuty to mocna poszlaka.
3. **Które zapytanie:** `pg_stat_activity` pokazuje zapytania wykonujące się teraz, ich czas i `wait_event`. `pg_stat_statements` pokazuje, które zapytanie najbardziej zwiększyło `total_exec_time` i `shared_blks_read`.
4. **Dlaczego:** `EXPLAIN (ANALYZE, BUFFERS)` z prawdziwymi parametrami. Szukaj miejsca, gdzie szacowane `rows` najbardziej rozjeżdża się z `actual rows`, i sprawdź, czy zawężający warunek trafił do `Index Cond`, czy tylko do `Filter`.
5. **Po stronie aplikacji:** czasy wykonania jobów i liczba przetworzonych wierszy na przejście. Nagły skok czasu do limitu przy prawie zerowej liczbie wierszy wskazuje na pętlę z części 7.

## 11. Wnioski

- **Porównanie krotek `(a, b) > (x, y)` stosuj tam, gdzie `a` naprawdę się zmienia**, np. paginacja `ORDER BY created_at, id`. Jeśli `a` jest i tak ustalone równością, napisz `a = x AND b > y`. Wynik będzie ten sam, a planer oszacuje go dobrze.
- **Planer szacuje krotkę wyłącznie z pierwszej kolumny.** Pozostałe kolumny nie wpływają na oszacowanie, a para „od–do” z krotek nie jest rozpoznawana jako przedział.
- **Plan może się zmienić bez deployu**, bo autoanalyze regularnie odświeża statystyki. Zapytania na gorącej ścieżce trzeba monitorować pod kątem czasu wykonania, nie tylko błędów.
- **Pętla „czytaj, aż dogonisz ruchomy cel” zamienia wolne zapytanie w stałe obciążenie.** Gdy strona trwa dłużej niż odstęp między nowymi danymi, pętla nigdy się nie kończy przed limitem czasu.
- **Przy poprawce udowodnij, że wynik się nie zmienia** (rozpisanie warunków jak w części 8) i porównaj plany przed i po na tych samych parametrach, najlepiej na produkcyjnych statystykach. Lokalna baza ma inne statystyki i może wybrać inny plan.
- **Przeszukaj kod pod kątem tego samego wzorca.** Inne zapytanie z tą samą krotką może przestawić się przy następnym autoanalyze.

## Źródła

- Dokumentacja PostgreSQL: [Row Constructor Comparison](https://www.postgresql.org/docs/current/functions-comparisons.html#ROW-WISE-COMPARISON)
- Dokumentacja PostgreSQL: [Statistics Used by the Planner](https://www.postgresql.org/docs/current/planner-stats.html) i [How the Planner Uses Statistics](https://www.postgresql.org/docs/current/planner-stats-details.html)
- Dokumentacja PostgreSQL: [Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- Dokumentacja PostgreSQL: [The Autovacuum Daemon](https://www.postgresql.org/docs/current/routine-vacuuming.html#AUTOVACUUM)
- Kod PostgreSQL: `rowcomparesel` w `src/backend/utils/adt/selfuncs.c`; łączenie warunków w przedział w `clauselist_selectivity` (`src/backend/optimizer/path/clausesel.c`)
