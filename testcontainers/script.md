# Bevezetés a Testcontainers használatába Spring Boot alkalmazásban

Integrációs tesztek, vagy fejlesztés közben az alkalmazás futtatásakor
gyakran szükség van adatbázisra, vagy más háttérszolgáltatásra, mint pl.
cachere vagy message brokerre. Használhatjuk
a saját számítógépünkön lévőt, azonban ennek
telepítése, frissítése körülményes lehet, különösen akkor,
ha esetleg egyszerre két különböző verzióját szeretnénk 
használni. Relációs adatbázis esetén használhatunk
pl. in-memory H2 adatbázist, ezzel azonban
kompatibilitiási problémák merülhetnek fel.
A virtualizáció, majd a konténerizáció ezen sokat segített.

A Testcontainers használatával a háttérszolgáltatásokat tartalmazó konténereket kódban tudjuk definiálni,
és kezeli ezek életciklusát, azaz a megfelelő időben elindítja és leállítja őket.

A Spring Boot a 3.1-es verziótól kezdve különösképp támogatja a Testcontainers használatát.
Nem egyedülálló módon, a Quarkusban is megjelenik Dev Services néven.

## Testcontainers használata tesztesetekben

Vegyünk egy létező Spring Boot 3 alkalmazást, mely alkalmazottakat kezel,
CRUD műveleteket lehet végezni REST API-n keresztül. Az adatokat egy Postgres
adatbázisban tárolja. Létezik egy `EmployeesIT` integrációs
teszt osztály, melyben a tesztesetek a teljes alkalmazást elindítják,
és `WebTestClient` segítségével REST szolgáltatásokat hívnak. Ezek futtatásához szükség van
egy adatbázishoz. Ezt a Testcontainers segítségével fogom elindítani.

Ehhez először felveszem a `spring-boot-testcontainers` és
`postgresql` függőséget.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
```

Valamint azért, hogy a konfiguráció és az elindított konténerek is újrafelhasználhatóak legyenek
a tesztesetek között, a Testcontainers konfigurációját kiteszem egy `TestcontainersConfiguration`
osztályba. Ellátom a `@TestConfiguration` annotációval, valamint létrehozok egy `PostgreSQLContainer`
típusú beant. Konstruktor paraméterben az image nevét adom meg, a reprodukálhatóság miatt pontos verziószámot használom.

Itt egyéb műveleteket is el tudnék végezni, pl. környezeti váltózót beállítani, parancsot futtatni, vagy fájlt másolni.

```java
@TestConfiguration(proxyBeanMethods = false)
public class TestcontainersConfiguration {

    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>("postgres:16.3");
    }

}
```

El kell látni a `@ServiceConnection` annotációval. Ekkor ugyanis létrejön egy `ConnectionDetails`, 
itt egy `JdbcConnectionDetails` példány, mely tartalmazza a kapcsolódási paramétereket, mint
pl. a konténer portja, mivel ezt a Testcontainers random határozza meg. Ez felülírja
az `application.properties` fájlban lévő értékeket.

Az osztályt behivatkozom az integrációs tesztben.

```java
@Import(TestcontainersConfiguration.class)
```

A teszt futtatásakor a logban látható, hogy elindít egy konténert, és a _Service_ fülön is látszik egy Postgres és egy másik konténer. 
(Ez szabadítja fel a teszt futtatása után az erőforrásokat.) A teszt után eltűnnek a konténerek.

Két teszteset futott le, és mindkettő ugyanazt a konténert használta. Ugyanezt a konfigurációt importáló másik teszt osztályokban
lévő tesztesetek is ugyanezt a konténert használnák.

## Spring Initializr

A Spring Initializr is hasonló kódot generál, ha felvesszük a függőségek közé a Testconatiners-t.

De itt van még egy test application is.

## Test application

Ugyanis a teszt ágra is felvehető egy `main` metódussal rendelkező osztály, mely szintén megkapja a
Testcontainers konfigurációt, így ha elindítjuk, vele elindul az adatbázis is.

```java
public class TestEmployeesApplication {

    public static void main(String[] args) {
        SpringApplication.from(EmployeesApplication::main).with(TestcontainersConfiguration.class).run(args);
    }
}
```

A Services fülön látható a futó konténer, valamint a http fájlból kéréseket indíthatunk az alkalmazás
felé (`POST`, `GET`).

Ha leállítjuk az alkalmazást, eltűnik a konténer.

## Futtatás parancssorból

Ezt használva, az alkalmazás Git clone után elindítható egy olyan gépen is, ahol csak egy JDK és egy Docker van telepítve.

```shell
mvnw spring-boot:test-run
```

Hiszen a Maven Wrapper letölti a Mavent, ha szükséges, a Testcontainers pedig elindítja a szükséges konténereket.

## DevTools

A DevTools használatakor nem kell újraindítani az alkalmazást módosításkor. Ehhez fel kell venni a
DevTools függőséget.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
</dependency>
```

Azonban a módosításkor, mikor újra töltődik az Application Context, a régi konténer leáll és új konténer indul.
Ez orvosolható a `@RestartScope` annotáció használatával, ugyanis ekkor az adott bean nem kerül újra létrehozásra.

## Konténer megtartása teszeseteknél

Mi van, ha egy teszteset lefuttatása után meg szeretném vizsgálni az adatbázis tartalmát?
Ehhez az kell, hogy a Testcontainers ne állítsa le a konténert. Ehhez a HOME
könyvtáramban lévő `.testcontainers.properties` fájlnak tartalmaznia kell
a `testcontainers.reuse.enable=true` értéket, valamint a konfigurációban meg kell hívni a
`.withReuse(true)` metódust.

## Konténer megtartása az alkalmazás futtatásakor

Ugyanez a helyzet az alkalmazás esetén is, az újraindított alkalmazás ugyanazt a konténert használja, és
megmaradnak az adatok. Ekkor a `@RestartScope` is elhagyható.

# YouTube aláírás

Ez a videó bemutatja, hogy hogyan lehet a Spring Boot 3-ban használni a Testcontainers-t
háttérszolgáltatások futtatására, akár teszteseteknél, akár alkalmazás futtatásakor.

GitHub repo: https://github.com/vicziani/jtechlog-testcontainers
JTechlog: https://jtechlog.hu
LinkedIn: https://www.linkedin.com/in/viczianistvan/

- Háttérszolgáltatások
- Testcontainers
- Testcontainers használata tesztesetekben
- Spring Initializr
- Futtatás parancssorból
- DevTools
- Konténer megtartása teszeseteknél
- Konténer megtartása az alkalmazás futtatásakor