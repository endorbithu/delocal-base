# DelocalBase

Module/Microservice hibrid keretrendszer + Admin felület simple-crud-dal és datatable-lel + általános funkciók weboldalakhoz.

**Features (részletesen lentebb, vagy adott package readme-ben kifejtve):**

1. Modularitás / microservice jellegű működés biztosítása `app_modules/` mappa (module deploy, module assets deploy, API
   logika stb.)
2. `app_modules/ExampleModule` biztosítása, benne a feature-rökre példa.
3. Admin logika biztosítása (admin user entitás + szerepkörök) `/nimda` URL
4. Admin értesítések (+ értesítés delegálása admin userhez)
5. Admin activity log (Adminon keresztül való mentés + törlés logolása DB-be, admin-userhez kötve, a változást is rögzítve)
6. datatable (lsd. még saját README.md)
- Eloquent entitás egyszerű kilistázása, DB join kapcsolatokat is lekezeli (N+1 hiba elkerülésével)
- Összetett keresés az oszlopoknál
- CSV export (akár nagy méretű adathalmazt is, mivel háttérműveletként fut(hat), tehát nem timeoutol)
- Custom action gombok létrehozása: kijelölt sorok (ID-k) a megadott POST végpontra küldése

7. simple-crud (lsd. még saját README.md)
- CRUD eszköz Eloquent entitásokhoz (datatable package-t is használja)
- multilanguage adatbázis szinten (egy mezőn belül több nyelv), tehát a __(...) mellett tudunk DB-ből jövő adatokat is
  az aktuális nyelven kiíratni
- képfeltöltés /galéria kezelés (előre definiált átméretezéssel, galéria, egy mezőben a képek locációi + mappába menti a
  képeket)
- file / multifile feltöltés

8. Teljeskörű config logika admin szerepkör korlátozással (admin GUI-ban is módosítható config, `config('module....')
   helperrel is elérjük)
9. Multinational működés biztosítása (language, timezone, currency)
10. Adat migráció, verziós (data_migrations)
11. Segéd middleware-ek (pl.: `NotAvailableToNull`)
12. Cast-ok (pl.: `JsonNice`)
13. `StatusBarException` + Handler.php hibakezelés módosítása, hiba azonosító kiírása productionön stb.
14. Egyszer/X-szer használatos linkek  (lejárati idővel)
15. Privacy és Terms logika biztosítása több scope lehetőséggel (verziós, fájl feltöltős)

Tartalom:

- I. INSTALL
- II. SETUP
- III. HASZNÁLAT

## I. INSTALL

0km-es projektnél laravel install:

```
composer create-project laravel/laravel:^X.Y htdocs
```

Utána a `composer.json`-ben be kell állítani a packagist-et:


És jöhet is a DelocalBase:

``` 
composer require .../delocal-base
```

## II. SETUP

```
php artisan vendor:publish --force
```

- datatable
- simple-crud
- delocal-base

```
php artisan migrate
```

Az `config/app.php`. `timezone` értéke legyen `UTC`, mivel konvencionálisan a DB datetime mezőit illik UTC_ben tartani.
A schedule (cron) -nak és logolás timezone beállításához adjuk hozzá a `config/app.php` hoz a `schedule_timezone` kulcsot, és 
adjuk meg pl. `Europe/Budapest`, és akkor a beállított cron értékek + logolás időpontok a Budapesti időzóna szerint lesznek.


#### Alap admin user hozzáadása:

```
php artisan db:seed --class=InsertAdminRoleAndAdmin
```

### Example Module + hozzá example adatok (fontos, mert galibát okozhat, ha nem vesszük fel):

```
config/app.php

'providers' => [
  ...
  ExampleModule\Providers\ServiceProvider::class,
],
```

```
composer.json

 "autoload": {
        "psr-4": {
            ...
            "ExampleModule\\": "app_modules/ExampleModule/src/"
```

```
composer dump  
php artisan migrate  
php artisan datamigrate  
```

## III. HASZNÁLAT

**Kapcsolódó package-ek:**

- Package-ek
  - datatable (lsd. még saját README.md)
  - simple-crud (lsd. még saját README.md)

- 3nd party package-ek
  - intervention/image - https://github.com/intervention/image => Intervention - teleskörű képkezelő eszköz (Simplecrud
    behúzza)
  - opcodesio/log-viewer - https://github.com/opcodesio/log-viewer => GUI-s log kezelés (opcionális)
  - enlightn/enlightn - https://github.com/enlightn/enlightn => A Laravel Tool To Boost Your App's Performance &
    Security (opcionális)

**FONTOS: A package-knél nézzük át a deploy során létrejött config fájlokat!**

## 1. Moduláris/Microservice hibrid működés

### Klasszikus moduláris elrendezés, új modul hozzáadása

- A modulok alap esetben formailag klasszikus modulok (package jelleggel: külön namespace-t kapnak, lsd. ExampleModule)
- A modulok tartalmilag viszont jellempzen kisebb egységet képviselnek, egy dologért felel, vagy egy-egy nagyon szorosan összefüggő dolgokért egységként. (Domain Driven Design segít elhatárolni ezeket az egységeket.)
- A `config('delocalbase.modules_directory')` ban meghatározott mappába kell az egyes modulokat (package-eket)
  elheyeznünk (az ExampleModul automatikusan belekerül). Alapértelmezetten ez az `/app_modules` mappa.
- composer.json -ben fel kell venni az  `"autoload": { "psr-4": {` alá az adott modul `src` mappáját.
- `config/app.php` -ban pedig az adott modul `ServiceProvider`-ét kell hozzáadni a  `'providers' => [` -hez
- Ezután a `php artisan module:deploy` gondoskodik az egyes modulok deploy-áról (config, képek, stb) (a a `php artisan vendor:publish` -ra
  épül)
- FONTOS: ami bent van mappa (module) az `app_modules/` mappában, azt mindeképp vegyük fel `composer/config/app.php` ba,
  mert hibára futhat adott esetben.
- FONTOS: a route fájlokban ÉS a controllerek `__construct()` methodjában mindig valid adatok legyenek, mert különben
  elszáll a route cache, meg jó eséllyel az egész route-olás

### Microservice jellegű működés ==> microservice képesség

- Az egyes moduloknak egymástól szigorúan el kell különülniük:
  - Egymás `config()` értékeit ne hivatkozzák meg
  - Egymás `Event` és `Eloquent event` -jeire NE IRATKOZZANAK fel
  - Két modul egymással csak a `DelocalBase\Api::make()` és a `DelocalBase\Events\ApiEvent` eventtel kommunikálhat:
  - Külvilágnak (is) szánt `Event` mindig a `DelocalBase\Events\ApiEvent` be legyen "becsomagolva", ahol stringben kell
    megadni az event nevét:
    - `event(ApiEvent('eventName', $datas))` a `$datas` csak `scalar`+`NULL`, vagy ezeket tartalmazó `array` lehet!
    - Ezeket az eventeket el lehet kapni természetsen saját modulon belül is, de "broadcast" módon ki lesz küldve
      minden modulnak is (a laravelen belüli modulok simán elkapják, a microservice-ek HTTP-n
      a `DelocalBase\Jobs\TriggerOutcomingApiEvents` -nak köszönhetően), és bárhol is
      vagyunk (scope-on belül, vagy microservice) a Listenereknek a
      `DelocalBase\Events\ApiEvent`-re kell feliratkozni, és a `ApiEvent::$name` -et
      kell vizsgálni, hogy fel kell-e dolgozniuk, vagy sem.
  - "Külvilágnak" (is) szánt, service methodoknak csak `scalar` +`NULL`, vagy ezeket tartalmazó `array` lehet
    paraméternek megadva.
  - Ugyanígy a külvilágnak (is) szánt service-ek visszatérési értéke is csak `scalar`+`NULL`, vagy ezeket
    tartalmazó `array` lehet.
  - A modulok egymás szolgáltatásait ne direktbe vagy `app(...)` / `App::make(...)` módon hívják meg,
    hanem így: `DelcalBase\Api::make('OtherExampleModule\\Api')` === `DelcalBase\Api::make('OtherExampleModule')` segéd
    method-ot használva. Ha a service osztály a  `OtherExampleModule\\Api`, akkor a végéről az `\\Api` -t elhagyhatjuk. NE
    A `::class` magic constant-tal adjuk meg a szolgáltatás (inteface) nevét, hanem "pötyögjük" be.
    - a) Ha ugyanazon laravel alatt van a cél modul, akkor automatikusan `App::make(...)` -kel fogja intézni, hiszen
      rálátunk a namespace-re direktbe
    - b) Ha nem ugyanazon laravel alatt van a cél modul (= microservice), akkor
      automatikusan a `config('delocalbase.service_namespace_endpoints')` -ban definiált végpontra kérdez be:

```
  'service_namespace_endpoints' => [
      'OtherExampleModule' => 'https://otherdelocalproject.loc',
  ],
```

Az `OtherExampleModule` namespace-t megvalósító microservice is célszerű, ha egy Laravel, amely
tartalmazza a `delocal-base`
package-t és struktúrát. De a lényeg, hogy ez nem köt minket. bármilyen HTTP API hívást fogadni tudó alkalmazás lehet, bármilyen programozási nyelven írva, 
a lényeg, hogy fel tudja dolgozni a delocalbase által összerakott JSON tartalmú HTTP hívást, és formailag megfelelő választ adjon vissza.

Ha delocalbase-t tartalmazó Laravel a távoli microservice, akkor ez fog lezajlani:
 a `DelocalBase\Api` lefordítja a bejövő HTTP üzenetet, Service::method-ra:  
`https://delocalproject.loc/serviceapi/OtherExampleModule/Api/amethod BODY json($params...)`
=> `OtherExampleModule\Api::amethod($params...)`
és a visszatérési érték meg visszautazik HTTP üzenetben.

**Összefoglalva:**  
**Service**  
`$val = DelcalBase\Api::make('OtherExampleModule\Api')->amethod($param1, $param2 ..)`

- a) HA ugyanazon a laravel-en van a `OtherExampleModule` namespace, akkor automatikusan
  => `App::make('OtherExampleModule\Api')->amethod($param1, $param2 ..)` kapja meg a `$val`
  változó
- b) HA külső microservice-ben, akkor:

1. HTTP üzenet => OtherExampleModule
   microservice-nek ( `https://delocalproject.loc/serviceapi/OtherExampleModule/Api/amethod`
   BODY: `json_encode($datas)`)
2. Microservice-en
   belül: `$return = App::make('OtherExampleModule\Api')->amethod($param1, $param2 ..)`
3. HTTP üzenet vissza `$return` értékkel amit megkap az 1. pontban lévő `$val` változó.

**Event + Listener**  
`event(new DelocalBase\Events\ApiEvent('event_neve', $datas))`

- a) HA ugyanazon a laravel-en van egy erre az `ApiEvent->name == event_neve` -re feliratkozott `Listener`, akkor simán
  elkapja az `ApiEvent`-et és csinálja amit kell
- b) HA külső microservice is egy feliratkozó, akkor (mint minden microservice) HTTP-n megkapja és triggerelve lesz ott
  is az  `ApiEvent` `event_neve` + `$datas`-szal.

A lényeg, hogy mindent a `DelocalBase\Api` osztálya + segédeszközök (route, Job stb) intéz, nem kell az egyes moduloknak
tudniuk, hogy a másik hol van,  
így később is ki tudunk szervezni a modult microservice-nek, anélkül, hogy a többi modul tudna róla, tehát, hogy bármit
át kelljen írni a kódban. De FONTOS, hogy ehhez be kell tartani a fent említett elhatárolásokat.
(Microservice kiszervezésnél csak a `config('delocalbase.service_namespace_endpoints')` -ban kell megadni az adott
namespace-hez az URI-t.)

TODO: Később, ha valóban kiszervezésre kerülnek microservicebe modulok,
akkor pl. RabbitMQ -t lehet beiktatni a két laravel végpont `DelocalBase\Api`-ja közé, hogy megbízhatóbb legyen a
kommunikáció, (most pl. az `DelocalBase\Events\ApiEvent` -ek Queue Job segítségével vannak CURL HTTP üzenetekként
kiküldve, és a
microservice-ben `/eventapi/Eventname` route-ba fut be, és ott triggerelődik a "helyi" laravelben ugyanaz
a `DelocalBase\Events\ApiEvent` event a küldött adatokkal.

Ezzel az egész kialakítással ezen kívül azt is nyerjük, hogy ebben microservice-es paradigmában,
jobb minőségben, magas decoupling szinten fejlesszük le a modulokat.

## 2. ExampleModule biztosítása

- Modul struktúra kidolozása
- Datataable és SimpleCrud feature-jeinek bemutatása
- admin menüpont + oldal létrehozása
- stb. a fent felsorolt feature-ök bemutatása

## 3. Admin logika biztosítása

(admin user (Authenticatable) entitás + szerepkörök, CRUD logika (SimpleCrud package))

- `/nimda` végpont azonnal elérhető az admin felület
- `super` admin szerepkör
  létrehozása `php artisan db:seed --class=InsertAdminRoleAndAdmin` -del
- lehetőség, hogy egy-egy modulból tudjunk admin menüpontot és admin oldalt létrehozni (lsd ExampleModule)

## 4. Admin értesítések (+ értesítés delegálása admin userhez)

- `\DelocalBase\Events\NotifyAdmin` eventet kell triggerelni
- email kiküldést is ki lehet jelezni (arra nincsen listener, de meg lehet csinálni saját modulban)
- admin: Értesítések (olvasatlan) alatt sorolja fel,
- személyre (adminra) szabottan számolja/jelzi meg az olvasott/olvasatlan értesítéseket)
- adminban hozzá lehet az egyes értesítésekhez rendelni az értesítésért felelős admin usert.

## 5. Config

Teljeskörű config logika (admin GUI-ban is módosítható config, admin szerepkör korlátozással)

Minden esetben az `{adott_modul}/config/module.php` -ban a `[ 'adott_modul' => [...]]` ban kell felvenni új elemet,
`|` (pipeline) jellel kell jelezni (az array keyben), hogy az config érték további kezelése az admimból történjen,
tehát a fájlban megadott érték csak kezdőérték lesz, a fájlban később hiába módosítjuk az értéket a DB-ben szereplő
érték fog számítani)

```
return [
    'site' => [
       'vat_percent|Áfa %' => 27.0,
     ]
]
```

A kódban a pipeline+leírás nélküli kulccsal tudunk rá hivatkozni!
`config('module.site.vat_percent')`

Ahol nincs pipeline, azt az értéket csak az {adott_modul}/config/module.php fájlban tudjuk módosítani, (adminban csak az
aktuális értéket tudjuk megnézni)

Bármely adminban és/vagy `{adott_modul}/config/module.php` ban történő módosítás során
a `LARAVEL_ROOT/config/module.php`
fájl frissül. Nem debug környezetbenm ilyenkor törölnünk kell a cache-t is.

## 6. Multinational működés biztosítása

### SETUP

1. Kell beállítanunk a `currencies` táblában 1db `is_base=1` currency-t, ez lesz az alapértelmezett (fallback)

2. User mezőket beállítani db-ben:

- `timezone` (string) fallback=config('app.fallback_user_timezone')
- `currency` (string) fallback=HUF,
- `language` (string) fallback=hu

ha valamelyik mezőt nem csináljuk meg, vagy az értéke null, akkor ott a fallback-et fogja használni.

3. Ha van több nyelv, akkor a nyelvi fájlokat a `project_root/resources/lang/...` -ba kell tenni!

Inicializálás: a guardhoz tartozó user entitásnak kell beállítani 3 attribútumot, amik paraméterezik a multinacionális
logikát. Ha felmerül multinalcionális
működés, akkor ezeket a regisztráció során / profil szerkesztésnél adhatják majd meg, addig is mindenkinek az
alapértelemezett (fallback) értékek kerülnek be:

- `timezone` (alapértelemzett: Europe/Budapest)
- `language` (alapértelemzett: hu)
- `currency` (alapértelemzett: HUF)

Ezek az értékek bejelenetkezés után bekerülnek a aktuális `session` -be és ez alapján írja ki az oldal a
szövegeket, árakat, dátum/időket. A nyelvhez meg a currency-hez ha szükséges ki lehet tenni frontend kapcsolókat,
hogy tudja módosítani a user ezeket az értékeket menet közben is a session-ben, ha váltani akar pl angol nézetre.

### Timezone

Az adatbázisban a DateTime adatok UTC-ben vannak tárolva (`config('app.timezone') == 'UTC'`), kivéve speciális esetben,
pl a CRON időzónájához mentünk időpontot.

Ha valamelyik (UTC) datetime mezőt szeretnénk a user által beállított (vagy ha üres/guest: fallback)
timezone-ban mutatni, akkor az Eloquent Model-ben az adott mezőnél állítsuk be a
`DatetimeInUserTimezoneCarbon::class` cast Osztályt, és bárhol ki akarjuk íratni, ezt figyelembe fogja venni.

```
  protected $casts = [
         'created_at' => DatetimeInUserTimezoneCarbon::class,
```  

**Mentés**  
Itt van egy olyan helyzet, hogy a `HasAttributes` traitben a logika leCache-eli a get() -ben transformált datetime-ot
mergeAttributesFromClassCasts() környékén  
egy string-be, és valamiért meghívja a `DatetimeInUserTimezoneCarbon::set` fg-t így UTC-t csinálna belőle, annak
ellenére, hogy
amúgy csak `get()`-et szeretnénk, úgyhogy a datetime mentéseknél a timezone-t (amik nem automatikusak) ezzel a fg-nyel
kell beállítani a logikában:

```
     $user->valamilyen_datetime = app(MultinationalInterface::class)->traverseDatetimeToUTC($datetime);
```

itt a `$datetime` lehet: Carbon, DateTime, string típusú is, Carbon objektummal fog visszatérni,
ha nincs beállítva a usernél/guest van, akkor meg `fallback_user_timezone` nak veszi az adott időpontot, és abból
számolja ki az
UTC-t.

Ami automatikus datetime mentés (`created_at` stb.), az eleve UTC-be menti le a config beállítás szerint, azzal nincs
teendőnk.

### Language

A felületen minden szöveget így írassunk ki:

```
@lang('Valami sima szöveg')
@lang('Állásportálunkon <b class="text-secondary"> :nr db állásajánlat</b> között válogathatsz!', ['nr' => 23748])

throw new StatusBarException(__('Nincs elég egyenleged!'));
```

Ha később felmerül más nyelv, akkor ezeket a sringeket kell csak kigyűjteni valami scripttel, és
szótárfájlt csináltatni (en.json, de.json stb) és a `{project_root}/resources/lang` mappába helyezni.
Aztán ha be van állítva a usernek a `language` mezője, vagy frontend kapcsolóval beállította
arra a nyelv-re GET param-mal:

```
?language=de
```

akkor bekerül a nyelv a sessionbe, és a laravel teszi a dolgát, a kiválasztott nyelv fog megjelenni, egyébként meg marad
a fallback (magyar) nyelv.

### Currency

Itt is megvizsgálja, hogy van e a User-nek `currency` mezője, ha nincs, akkor default currency lesz.

+ GET parammal `?currency=USD` is lehet dinamikusan állítani, hogy melyik currency kerüljön a sessionbe.  
  Utána ezekkel a fg-kel tudunk operálni, akkor is használjuk ezeket a függvényeket, ha csak egy pénznem van, mert ezzel
  tudjuk formázni is ezres tagolás tizedesek száma (`Price:: facade`), és később nem kell átbogarászni az egész kódot,
  ha bejön többféle pénznem.

```
Price::getConvertedPriceWithSign($amount, $toCurrency = null, $decimal = null, $longName = false): string //$toCurrency alapértelezetten: a sessionben lévő currency
Price::getConvertedPrice($amount, $toCurrency = null, $decimal = null): float //$toCurrency alapértelezetten: a sessionben lévő currency
Price::getConvertedPriceToSave($amount, $fromCurrency = null): float //$fromCurrency alapértelezetten: a sessionben lévő currency
```

## 7. DataMigrate modul

Lehetőségünk van adatot insert/update/delete -ni az adatbázisban, ehhez bármelyik modulon belül
a `app_modules/{ModulNeve}/database/data_migrations/` mappában kell
létrehoznunk egy `.php` fájlt a fájlnév eleján szerepelnie kell az aktuális dátum időnek mindegy, hogy miéyen kötöjel
stb-vel elválasztva, de 12 szám legyen benne YYYmmddHHii => pl: 2022-12-12_09-08-01_InsertSomeConfig.php,
20221112_121110_ValamiDatamigrate.php.
Ez a fájl egy invokable anonymus class-al térjen vissza:

```
<?php
    return new class() {
        function __invoke() {
            //bármilyen logikával intézhetjük DB, vagy Eloquent stb.,
        }
    }
```

és utána  `php artisan datamigrate`
parancsra végignézi az összes modulnak a `database/data_migrations` mappáját dátum szerint növekvő sorrendben, és
amelyik fájl még nem lett feldolgozva, azt futtatja.
Sikeres feldolgozás után a `data_migrations` táblába regisztrálja, ez jelzi időbélyeggel, hogy már fel lett dolgozva.

**Rollbackelni nem lehet** ezért művelet elején mindenképp **vizsgáljuk meg**, hogy az adott adat bent van-e már,
törölve van-e már stb ne írja felül/ne legyen duplikáció, ne szálljon el mert nem találja stb.

`php artisan datamigrate fájl-neve` => csak az adott fájlt futtatja, ha lefutott már, ha nem.

## 8. Segéd middleware-ek

### NotAvailableToNull

- Ha egy input mezőnek az az értéke, hogy `n/a` akkor az automatikusan `NULL` értékre fog változni a requestben

### DelocalBase modul által használt middleware-ek:

- IPCheckerForXhr
- ModuleConfigChecker
- RedirectGuestToAdminLogin
- ServiceApiIpValidator
- SetMultinationalDatas

## 9. Cast-ok

### JsonNice

- json <=> array conversion, csak itt a json string tagolt lesz, és az ékezetes karatkerek is rendesen látszódni
  fognak `JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE`

### DatetimeInUserTimezoneCarbon

lsd. Multinational

## 10. StatusBarException + Handler.php hibakezelés módosítása

### StatusBarException

Bárhol (nem console) dobunk egy `throw StatusBarException(__('Üzenet'))` :

- redirect previous page VAGY frontpage (ha nem internal oldalról jövünk)
- flash error üzenetben az `$e->getMessage()`
- `statusbar.blade.php` biztosítása, ami értelemszerűen kiírja a success warning error üzeneteket, ez persze felülírható

### Handler.php hibakezelés módosítása

Exception logolásnál az error ID mellett a `User agent` és a `request` is mentésre kerül:
(a `passw...` -vel kezdődők nem + a hosszú request paraméterek végét levágjuk. pl:

``` 
[2023-04-28 12:01:26] production.ERROR: ERROR ID: 20230428_120126_644bb596bdde939
Request path: GET /simplecrud/Job/10238543/image
Request datas: {"file":”valami.jpg”}
User agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.0.0 Safari/537.36
Symfony\Component\HttpFoundation\File\Exception\FileNotFoundException: The file "/var/www/cvdb/cvdb.hu/storage/valami.jpg" does not exist in /var/www/cvdb/cvdb.hu/vendor/symfony/http-foundation/File/File.php:36
```

## 11. Egyszer/X-szer használatos linkek  (lejárati idővel)

Hasonló, mint a signed URL

## 12. Privacy és Terms logika biztosítása

több scope lehetőséggel (verziós, fájl feltöltős)
Az adminban benne van egyértelműen, az oldalon úgy kell kialakítani a routeokat, hogy a

- `/terms` => mutasson a legfrisebb terms rekordra
- `/privacy` => mutasson a legfrisebb privacy rekordra

## 13. 3nd party package-ek

- opcodesio/log-viewer - https://github.com/opcodesio/log-viewer
- config/log-viewer.php
  - Middleware-nek ne felejtsük el az 'admin' -t megadni!

  + egyéb config beállítások egyértelműek

