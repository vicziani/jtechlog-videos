# Események a Spring Modulith-ban

A Spring Modulith projekt segítségével modulokból álló, jól struktúrált alkalmazást lehet felépíteni. 

A legfelsőbb szintű csomagokat tekinti moduloknak, és a tesztek futtatásakor ezek kapcsolatait ellenőrzi.
Például megvizsgálja, hogy egy modul a másik modulnak csak az API csomagját hívja-e, valamint
azt is ellenőrzi, nincs-e a modulok között körkörös függőség.

Modulok közötti kommunikációra eseményeket használva laza lesz közöttük
a kapcsolat, feloldhatóak a körkörös függőségek, sőt a tesztelés is egyszerűbbé válik.
A Spring Framework alapból tartalmaz eseményküldést és fogadást, a Spring
Modulith ezt kiegészíti, biztosabbá teszi a hiba- és tranzakciókezelést, valamint képes
ezeket az eseményeket más service-ek felé is elküldeni valamilyen message brokeren keresztül.

Adott egy két modulból álló alkalmazás, mely az alkalmazottakat, és a hozzájuk kapcsolódó
képzettségeket tartalmazza. Az alkalmazottakat kezelő modul az alkalmazott törlésekor
dob egy eseményt az `EmployeeService` `deleteEmployee()` metódusában.

Az eseményt a képzettségeket kezelő modul fogadja
az `AcquiredSkillsService` `handleEmployeeHasBeenDeletedEvent()` metódusában, és törli az
alkalmazotthoz kapcsolódó képzettségeket.

## Tranzakciókezelés

Ez alapértelmezetten szinkron módon fut, abban a tranzakcióban, amelyben az eseményt 
eldobó metódus is fut.

Ezért érdemes rátenni az `@Async` annotációt, mely külön szálon futtatja, valamint
a `TransactionalEventListener` annotációt, mely azt biztosítja, hogy az előző tranzakció
commitja után fusson le, valamint a `@Transactional(propagation = Propagation.REQUIRES_NEW)`
annotációt, ami azért felelős, hogy saját tranzakciót indítson.

Ehelyett felveszek egy új függőséget:

```xml
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-events-api</artifactId>
</dependency>
```
És a három annotáció
helyett használhatom a Spring Modulith `@ApplicationModuleListener` annotációját,
mely önmaga tartalmazza mindhárom másik annotációt.

## Event Publication Repository

Ekkor még mindig megtörténhet az, hogy az esemény feldolgozása közben hiba lép fel,
és az esemény elveszik.

Ennek kivédésére az Event Publication Repository az
eseményt kiírja adatbázisba is. Ehhez felveszek egy függőséget,

```xml
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-jpa</artifactId>
</dependency>
```

Ha most létrehozok, majd törlök egy felhasználót, így küldök eseményt, 
akkor látható, hogy bekerül az `event_publication` táblába is,
kitöltött `completition_date` mezővel.

Az alkalmazásba már felvettem a Spring Boot Chaos Monkey projektet, mely segítségével
hibát tudok tenni a rendszerbe. Az Actuatorjának használatával beállítom, hogy
az `AcquiredSkillsService` `handleEmployeeHasDeletedEvent()` metódusa dobjon `RuntimeException`-t.
Ekkor a táblába üres `completition_date` értékkel kerül be. 
Ezeket az alkalmazás indulásakor próbálja újra elküldeni, ha be van állítva a `spring.modulith.republish-outstanding-events-on-restart` property. 

## Esemény küldése message brokeren

Erre az eseményre más service-ek is kíváncsiak lehetnek, ezért egy 
Kafka topicba is el fogom elküldeni.
Ehhez fel kell venni a következő függőséget.

```xml
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-events-kafka</artifactId>
</dependency>
```

Valamint az eseményre rátenni a `@Externalized` annotációt.

Ekkor a Kafkában, melyet Testcontainerssel indítottam, létrejön egy
`EmployeeHasBeenDeletedEvent` topic, és az alkalmazott törlésekor megjelenik benne egy üzenet.