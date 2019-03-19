# Build Once, Run Anywhere oder: Konfiguration über Docker verwalten

Sie finden den Code zum Artikel auf
[GitHub](https://github.com/MichaelKaaden/dockerized-app/tree/master/Part-2-Build-Once-Run-Anywhere).

In Teil I dieser Artikelserie haben Sie gelernt, wie Sie Ihre Angular-App in ein
Docker-Image packen und in einem Container zur Ausführung bringen können.

## Motivation

Die meisten Angular-Apps werden Daten abfragen und/oder persistieren. Sie
benötigen also ein Backend, meist in Gestalt eines RESTful Web Service. Die App
spricht das Backend über einen URL an, der irgendwo in der App abgelegt sein
muss. Das Angular-Team [stellt sich vor](https://angular.io/guide/build), dass
Sie diesen in `src/environments/environment.ts` bzw.
`src/environments/environment.prod.ts` ablegen, beispielsweise unter dem Namen
`baseUrl.

Das klappt gut, solange Sie mit dieseb beiden Umgebungen auskommen. Gerade wenn
Sie in einem Team entwickeln, werden Sie jedoch mindestens vier Umgebungen
verwenden:

-   _Development_ zum Entwickeln,
-   _Testing_ für Integrations- und Systemtests sowie manuelle Tests,
-   _Staging_ für die Produktabnahme und schließlich
-   _Production_ für den Produktivbetrieb.

Jede dieser Umgebungen hat typischerweise ihr eigenes Backend und benötigt somit
einen spezifischen `baseUrl`. In der Realität werden Sie noch mehr zu
konfigurieren haben, etwa einen Identity Provider für die Autorisierung, doch
bringen weitere Konfigurationsoptionen keinen Mehrwehrt zu diesem Artikel hinzu,
weshalb ich mich auf den `baseUrl` beschränken möchte.

Sie sehen schon: Mit dem `environment.ts` und `environment.prod.ts` kommen wir
da nicht ganz hin. Natürlich können wir noch ein `environment.staging.ts`
definieren, doch das hilft uns nicht. Warum das so ist, sehen wir gleich, wenn
wir uns überlegen, welche Anforderungen wir an die Konfigurierbarkeit unserer
App haben.

## Anforderungen an die Konfigurierbarkeit

In der _Development_-Umgebung soll die App ganz normal mit `ng serve` bzw.
`ng serve --prod` laufen können. Wir brauchen also eine Lösung, die den
`baseUrl` auch ohne Docker zur Verfügung stellt.

In den anderen Umgebungen möchten wir den jeweils passenden `baseUrl` zur
Verfügung haben.

Eine wesentliche Einschränkung dabei ist: Wir dürfen nicht damit anfangen, nur
wegen eines jeweils anderen `baseUrl` ein Image je Umgebung zu erstellen. Wenn
ein Image in _Testing_ für gut befunden wurde, egal ob durch automatische oder
manuelle Tests, dann ist genau dieses Image dasjenige, das nach _Staging_ und
nach erfolgreicher Abnahme nach _Production_ wandern soll. Wir dürfen _kein_
neues Image erstellen, denn das könnte geringfügig anders sein als das
getestete, beispielsweise weil in der Zwischenzeit Ihre geschätzten
Admin-Kollegen automatisiert eine neue `Node.js`-Version auf Ihrem Rechner
eingespielt haben. Vielleicht haben Sie auch Ihre globalen`npm`-Pakete
aktualisiert. Oder ein Kollege hat stillschweigend einen Fix in die Sourcen
eingebaut.

Wenn aber das Image aus _Testing_ auch für _Staging_ und _Production_ verwendet
werden soll, heißt das, dass der `baseUrl` nicht Teil des Image sein darf,
sondern von außen über Docker konfigurierbar sein muss.

Zusammengefasst benötigen wir also eine Lösung, die uns in der
_Development_-Umgebung einen `baseUrl` auch ohne Docker zur Verfügung stellt,
die uns aber einen Weg ebnet, den `baseUrl` mit Docker zu konfigurieren.

## Konfigurierbarkeit herstellen
