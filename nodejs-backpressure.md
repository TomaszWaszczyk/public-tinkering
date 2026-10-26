# Backpressure w Node.js

W Node.js backpressure jest wbudowane w strumienie z `node:stream`. Odbiorca ma wewnętrzny bufor, a nadawca dostaje sygnał, żeby przestać dokładać dane, gdy ten bufor dojdzie do progu `highWaterMark`.

Nadawca czeka, aż odbiorca nadąży. Tempo nie jest z góry ustalone — wynika z tego, ile odbiorca jest w stanie przyjąć w danym momencie.

## Jak to działa

`Writable.write(chunk)` zawsze kolejkuje kawałek. Zwraca `true`, dopóki długość bufora jest poniżej progu, i `false`, gdy próg został osiągnięty. `false` oznacza „ten kawałek już jest w buforze, nie dawaj następnego”. Gdy wywołania `_write` opróżnią bufor poniżej progu, strumień emituje `drain` i nadawca może pisać dalej.

Domyślny próg to 16 KiB (16 obiektów w trybie obiektowym). `fs.createReadStream` ustawia 64 KiB.

Po stronie `Readable` ten sam próg sprawdza `push(chunk)`. Wartość `false` znaczy, że bufor odczytu jest pełny i `_read` nie zostanie wywołane ponownie, aż konsument odbierze dane.

`readable.pipe(writable)` spina to samo: gdy `write` zwróci `false`, źródło jest pauzowane, a po `drain` wznawiane. `pipeline` z `node:stream/promises` robi to samo i dodatkowo zamyka oba końce przy błędzie.

```ts
import { once } from 'node:events';
import { createReadStream, createWriteStream } from 'node:fs';
import { pipeline } from 'node:stream/promises';

await pipeline(createReadStream('wejscie.bin'), createWriteStream('wyjscie.bin'));

// ten sam mechanizm ręcznie, przy pętli async
for await (const chunk of createReadStream('wejscie.bin')) {
  if (!dest.write(chunk)) await once(dest, 'drain');
}
```

W gnieździe TCP sygnał schodzi niżej. Gdy proces nie czyta z deskryptora, bufor jądra się zapełnia, okno TCP maleje do zera i drugi koniec przestaje wysyłać. Przy odpowiedzi HTTP wolny klient w ten sposób zatrzymuje odczyt pliku, o ile odpowiedź jest podłączona przez `pipe` albo `pipeline`.

Sam `EventEmitter` progu nie ma. Listener na `'data'`, który woła `write` i ignoruje `false`, rośnie w pamięci bez ograniczenia.

## Przykład w tym backendzie

Na zewnętrznym API, gdy klucz `tek_` ma już dwa żądania w trakcie, kolejne dostaje odpowiedź 429 z nagłówkiem `Retry-After` i ma poczekać, aż zwolni się miejsce. Licznik trzyma slot do końca pracy handlera, także gdy klient się rozłączył, bo zapytanie do bazy i tak leci dalej.

Implementacja: `apps/backend/src/modules/external-api/interceptors/external-concurrency-limit.interceptor.ts`.

Limit to `DEFAULT_EXTERNAL_API_MAX_CONCURRENT = 2`, a `Retry-After` wynosi 1 sekundę.

## Pytania rekrutacyjne

### Podstawowe

- Co zwraca `write` i co wtedy zrobić? `false` znaczy, że bufor doszedł do `highWaterMark`, kawałek został przyjęty, dalszy zapis czeka na `drain`.
- Jaka jest domyślna wartość `highWaterMark` i czym różni się tryb obiektowy? 16 KiB albo 16 obiektów. Strumień plikowy czyta po 64 KiB.
- Czym się różni tryb płynący (`data` + `pause`/`resume`) od wstrzymanego (`readable` + `read()`)?
- Co robi `pipe`, a co dodatkowo `pipeline`? Oba stosują backpressure. `pipeline` sprząta strumienie po błędzie i przy przedwczesnym zamknięciu.

### Średnie

- Jak backpressure realizuje własny `Readable`? `push` zwraca `false`, a kolejne dane powstają dopiero w następnym `_read`.
- Gdzie jest hamulec w `Transform`? Dopóki nie wywołasz callbacku `_transform`, następny kawałek nie przyjdzie.
- Co się stanie, gdy nasłuchujesz `'data'` i nie sprawdzasz wyniku `write`? Bufor zapisu rośnie bez progu.
- Jak wolny klient HTTP zatrzymuje wysyłanie pliku? `ServerResponse` jest `Writable`; pełny bufor pauzuje źródło, a pod spodem okno TCP.
- Czym się różni `cork()` / `uncork()` od backpressure? `cork` skleja wiele `write` w jeden zapis do systemu, progu bufora nie pilnuje.

### Trudniejsze

- Jak ograniczyć kolejkę zadań w pamięci, skoro to nie strumień? Licznik pracy w toku albo bufor z progiem; nadmiar czeka albo dostaje 429 z `Retry-After`.
- Jak rozlać jeden odczyt na kilku odbiorców? Pauza musi uwzględniać najwolniejszego. Samo `pipe` na kilka celów tego nie uzgadnia.
- Skąd wiadomo, że backpressure nie działa? Rośnie sterta przy wolnym odbiorcy, a `writableLength` trzyma się powyżej `highWaterMark`.
