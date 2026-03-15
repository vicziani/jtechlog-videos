# ThreadLocal, Project Reactor Context és a Java 25 ScopedValue

## ThreadLocal használata

Előfordul, hogy az egyik metódusból értéket kell átadnunk egy másik metódusnak anélkül, hogy azt paraméterként adnánk át.
Lehet ez amiatt, mert a két metódus között nincs közvetlen hívás, és nincs ráhatásunk a közbülső
metódusokra.

Ilyen lehet pl. a bejelentkezett felhasználó,
a tranzakció, vagy a log MDC.
Ezeket gyakran környezetnek (contextnek) nevezik, pl.
Security Context, Transaction Context, Mapped Diagnostic Context.

A Servlet API-ra épülő keretrendszerek, pl. a Spring MVC
esetén kihasználható, hogy egy HTTP kérést egy
szál szolgál ki. Pl. a Spring is szálhoz kapcsolja
a bejelentkezett felhasználót, a tranzakciót, vagy a JPA
persistence context-et. Sőt, ha request scope-pal definiálunk egy bean-t,
az is ehhez a szálhoz kerül letárolásra. A szálat indító http kérést is bárhol le tudjuk kérdezni.

Erre megoldás a `ThreadLocal` osztály, mely egy olyan adatstruktúra, mely lehetővé teszi, hogy 
szálanként különböző értékeket tároljunk.

Sőt, a `ThreadLocal` használható akkor is, ha egy osztály nem szálbiztos, viszont
létrehozása túl költséges ahhoz, hogy minden használat esetén újra példányosítsuk.
Ilyen például a `Random` osztály, melyhez kapcsolódik egy `ThreadLocalRandom` osztály is.

A `ThreadLocal` elsősorban keretrendszer fejlesztőknek hasznos, de ismeretével
jobban megérthetők az említett mechanizmusok.

Deklarálok két metódust, a hívó neve legyen `processOrder()`, a hívott neve `saveOrder()`.
Az előbbi generál egy véletlen UUID-t, melyre a másodiknak szüksége van.

A metódusokat három szálon hívom párhuzamosan. Ehhez létrehozok egy thread pool-t,
és három `CompletableFuture` példányt. Ahhoz, hogy a hívások különböző sorrendben történjenek,
teszek bele egy véletlen várakozást.

A 19-es Java-tól az `ExecutorService` implementálja az `AutoCloseable` interfészt.

Majd bevezetem a `ThreadLocal`-t, melyben a `processOrder()` elhelyezi az értéket.
A `saveOrder()` pedig kiveszi.

Lefuttatva ugyanahhoz az szálhoz ugyanaz a request id kapcsolódik.

```java
package imperative;

import lombok.SneakyThrows;
import lombok.extern.slf4j.Slf4j;

import java.util.List;
import java.util.Random;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.stream.IntStream;

@Slf4j
public class ThreadLocalApplication {

    private final Random random = new Random();

    private final ThreadLocal<String> requestId = new ThreadLocal<>();

    static void main() {
        new ThreadLocalApplication().run();
    }

    @SneakyThrows
    private void run() {
        try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
            executor.invokeAll(
                    IntStream.range(0, 3)
                            .mapToObj(i -> Executors.callable(this::processOrder))
                            .toList()
            );
        }
    }

    @SneakyThrows
    private void processOrder() {
        String id = UUID.randomUUID().toString();
        log.info("process: {}, {}", id, requestId.get());
        requestId.set(id);
        Thread.sleep(random.nextInt(1000));
        saveOrder();
    }

    private void saveOrder() {
        String id = requestId.get();
        log.info("save: {}", id);
        // requestId.remove(); // Thread pool miatt
    }
}
```

Thread pool használata esetén, mivel a szálakat újrafelhasználja, a szálhoz tartozó érték megmarad.

Ha leveszem a thread pool méretét 2-re, akkor látszik, hogy egy korábbi szál értéke benne marad a ThreadLocal-ban. 
Ezért mindenképp figyelni kell arra, hogy a munka végeztével ürítsük a `ThreadLocal` értékét.

A `ThreadLocal` érdekessége, hogy úgy működik, hogy a `Thread` osztályban van egy
`ThreadLocal.ThreadLocalMap threadLocals` attribútum, mely a 
`ThreadLocal`-hoz tárolja egy egyedi map adatszerkezetben az értéket.

## Context használata Project Reactorban

Project Reactor esetén is tudunk az egyik metódusból értéket átadni a másik
metódusnak anélkül, hogy ezt paraméterként deklarálnánk.

Itt azonban nem támaszkodhatunk a szálakra, ugyanis a Project Reactor concurrency-agnostic,
ami azt jelenti, hogy nem erőltet ránk semmilyen párhuzamossági modellt. Ezért elképzelhető,
hogy bizonyos operátorok, mint például a `flatMap`, `publishOn`,  `subscribeOn` és `delayElement`
operátorok, szálat váltanak. Sőt, a Project Reactorra épülő Spring WebFlux 
minden kérést magonként ugyanazon a szálon szolgál ki.

A Project Reactor 3.1.0 verziójában jelent meg a `Context`, mely egy `Map`-szerű
adatszerkezet, mely streamenként képes különböző értékeket tárolni, melyekhez az
operátorok hozzáférhetnek.

Deklarálok két metódust, a hívó neve legyen `processOrder()`, a hívott neve `saveOrder()`.
Az előbbi generál egy véletlen UUID-t, melyre a másodiknak szüksége van.

A `contextWrite()` metódussal írok egy értéket, amit a `deferContextual()` metódus
hívásával tudom kiolvasni.

```java
@Slf4j
public class ReactiveContextApplication {

    static void main() {
        new ReactiveContextApplication().run();
    }

    private void run() {
        Flux.range(0, 3)
                        .flatMap(i -> processOrder())
                                .subscribe();
    }

    private Mono<Void> processOrder() {
        String id = UUID.randomUUID().toString();
        log.info("process: {}", id);
        return saveOrder()
                .contextWrite(Context.of("id", id));
    }

    private Mono<Void> saveOrder() {
        return Mono.deferContextual(ctx -> {
            log.info("save: {}", ctx.get("id").toString());
            return Mono.empty();
        });
    }

}
```

## Java 25 ScopedValue

A Java 25-ben jelent meg a `ScopedValue` osztály, melyet a `ThreadLocal` leváltására
vezettek be. A cél itt is az, hogy az egyik metódus értéket tudjon átadni a másik metódusnak anélkül,
hogy paraméterként definiálni kéne. 

A `ScopedValue` előnye, hogy jobban illeszkedik a structured concurrency-hez, és a virtual threadekhez, valamint az érték módosíthatatlan, és a hívás után automatikusan eltűnik,
nem kell explicit módon takarítani.

Ez a háttérben szintén a szálakra épül.

Deklarálok két metódust, a hívó neve legyen `processOrder()`, a hívott neve `saveOrder()`.
Az előbbi generál egy véletlen UUID-t, melyre a másodiknak szüksége van.

A `ScopedValue.where` hívásnál tárolom le az értéket, melyet a `get()` metódussal tudom kiolvasni. 

```java
@Slf4j
public class ScopedValueApplication {

    private final Random random = new Random();

    private final ScopedValue<String> requestId = ScopedValue.newInstance();

    static void main() {
        new ScopedValueApplication().run();
    }

    @SneakyThrows
    private void run() {
        try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
            executor.invokeAll(IntStream.range(0, 3)
                    .mapToObj(i -> Executors.callable(this::processOrder))
                    .toList()
            );
        }
    }

    @SneakyThrows
    private void processOrder() {
        String id = UUID.randomUUID().toString();
        log.info("process: {}", id);
        Thread.sleep(random.nextInt(1000));
        ScopedValue.where(requestId, id).run(this::saveOrder);
    }

    private void saveOrder() {
        String id = requestId.get();        
        log.info("save: {}", id);
    }
}
```
