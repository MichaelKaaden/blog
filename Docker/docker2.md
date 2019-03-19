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
da nicht ganz hin. Natürlich könnten wir noch ein `environment.testing.ts` und
ein `environment.staging.ts` definieren, doch das hilft uns nicht weiter, wie
wir gleich sehen werden.

## Anforderungen an die Konfigurierbarkeit

In der _Development_-Umgebung soll die App ganz normal mit `ng serve` bzw.
`ng serve --prod` laufen können. Wir brauchen also eine Lösung, die den
`baseUrl` auch ohne Docker zur Verfügung stellt.

In den anderen Umgebungen möchten wir den jeweils passenden `baseUrl` zur
Verfügung haben. Eine wesentliche Einschränkung dabei ist: Wir dürfen nicht
damit anfangen, nur wegen eines jeweils anderen `baseUrl` ein Image je Umgebung
zu erstellen.

Wenn ein Image in _Testing_ für gut befunden wurde, egal ob durch automatische
oder manuelle Tests, dann ist genau dieses Image dasjenige, das nach _Staging_
und nach erfolgreicher Abnahme nach _Production_ wandern soll. Wir dürfen _kein_
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

## Konfigurierbarkeit umsetzen

Um diese Anforderungen umsetzen zu können, benötigen wir einen Mechanismus, der
die Konfiguration der App erst zur Laufzeit lädt. Würden wir das
`environment.ts` für diesen Zweck nutzen, dann müsste die Konfiguration schon
beim _Build_ feststehen, was aber nicht sein darf, wie wir im vorangehenden
Kapitel festgestellt haben.

Eine Lösung dafür ist, die Konfiguration in eine Datei zu packen, die wir im
`assets`-Verzeichnis ablegen und von dort zusammen mit der App laden. Auf diese
Weise können wir diese Konfigurationsdatei beim Start des Containers beliebig
überschreiben. Wir ändern die Konfiguration dadurch zur _Laufzeit_. Wir sehen
gleich, wie wir das bewerkstelligen.

Hier ist die `src/assets/settings.json`-Datei, die wir zu diesem Zweck verwenden
werden.

```json
{
    "baseUrl": "http://localhost:5002"
}
```

Definieren wir ein `Settings`-Interface, das der Struktur der
Konfigurationsdatei entspricht:

```typescript
export interface Settings {
    baseUrl: string;
}
```

Jetzt brauchen wir noch einen `SettingsService`, den wir überall da injizieren
können, wo wir Zugriff auf die Konfiguration benötigen.

```typescript
import { Injectable } from "@angular/core";
import { Settings } from "../models/settings";

@Injectable({
    providedIn: "root",
})
export class SettingsService {
    settings: Settings;
}
```

Sehen wir uns den `SettingsInitializerService` an, der für das Laden der
`src/assets/settings.json` verantwortlich ist:

```typescript
import { HttpClient } from "@angular/common/http";
import { Injectable } from "@angular/core";
import { Settings } from "../models/settings";
import { SettingsService } from "./settings.service";

@Injectable({
    providedIn: "root",
})
export class SettingsInitializerService {
    constructor(private http: HttpClient, private settings: SettingsService) {}

    initializeSettings(): Promise<any> {
        return new Promise((resolve, reject) => {
            this.http
                .get("assets/settings.json")
                .toPromise()
                .then((response) => {
                    this.settings.settings = response as Settings;
                    resolve();
                })
                .catch((error) => reject(error));
        });
    }
}
```

Das war noch nicht weiter kompliziert. Spannend wird es, wenn wir die
Konfiguration aus dem `settings.json` laden wollen. Die App braucht die
Konfiguration, sobald der Browser sie startet. Dummerweise lädt die App die
Konfiguration aus dem `assets`-Verzeichnis und somit per HTTP(S)-Aufruf, somit
also asynchron. Wir brauchen also ein Mittel, um das Laden abzuwarten, bevor die
App startet. Glücklicherweise hat Angular dafür das Konzept des
`APP_INITIALIZER` eingeführt, das genau das leistet.

Passen wir also das `app.module.ts` mit dieser Erkenntnis folgendermaßen an, um
die Konfiguration zu laden:

```typescript
import { HttpClientModule } from "@angular/common/http";
import { BrowserModule } from "@angular/platform-browser";
import { APP_INITIALIZER, NgModule } from "@angular/core";

import { AppRoutingModule } from "./app-routing.module";
import { AppComponent } from "./components/app/app.component";
import { OneComponent } from "./components/one/one.component";
import { TwoComponent } from "./components/two/two.component";
import { SettingsInitializerService } from "./services/settings-initializer.service";

export function initSettings(
    settingsInitializerService: SettingsInitializerService,
) {
    return () => settingsInitializerService.initializeSettings();
}

@NgModule({
    declarations: [AppComponent, OneComponent, TwoComponent],
    imports: [BrowserModule, AppRoutingModule, HttpClientModule],
    providers: [
        {
            provide: APP_INITIALIZER,
            useFactory: initSettings,
            deps: [SettingsInitializerService],
            multi: true,
        },
    ],
    bootstrap: [AppComponent],
})
export class AppModule {}
```

Damit haben wir bereits die _Development_-Umgebung zum Laufen gebracht. Sie
können das mittels `ng serve` oder `ng serve --prod` leicht nachprüfen.

## Umgebung mit Docker konfigurieren

Wie erzeugen wir jetzt also pro Umgebung eine passende `settings.json`-Datei je
Container? Ich nutze dafür `envsubst`. Dieses liest ein Template, ersetzt darin
Umgebungsvariablen und schreibt das Ergebnis in eine neue Datei.

Hier ist das Template in Form der `src/assets/settings.json.template`-Datei:

```json
{
    "baseUrl": "${BASE_URL}"
}
```

Jetzt brauchen wir nur noch `docker-compose.yml` dahingehend anzupassen, dass es
`envsubst` verwendet, um zur Laufzeit die passende
`src/assets/settings.json`-Datei für die jeweilige Umgebung zu erstellen:

```yaml
version: "3"

services:
    web:
        image: dockerized-app
        env_file:
            - ./docker.env
        ports:
            - "8093:80"
        command:
            /bin/bash -c "envsubst '$$BASE_URL' < \
            /usr/share/nginx/html/assets/settings.json.template > \
            /usr/share/nginx/html/assets/settings.json && \ exec nginx -g
            'daemon off;'"
```

Wie Sie sehen, lädt `docker-compose` die Umgebung aus einer `docker.env`-Datei,
die folgendermaßen aussieht:

```bash
BASE_URL=http://some.official.server:444
```

Damit haben wir endlich alle Puzzleteile zusammen, um den Docker-Container mit
der gewünschten Umgebung zu starten. Sie brauchen lediglich die jeweils
passenden Umgebungsvariablen zu setzen. Die `dockerize.sh`- und
`redeploy.sh`-Skripte aus dem vorigen Teil funktionieren übrigens ohne Änderung
weiterhin.

Im letzten Teil der Artikelserie zeige ich Ihnen, wie sie die Buildumgebung über
die Projektlaufzeit im Griff behalten, auch wenn Sie Ihr System durch neue
Versionen der benötigten Werkzeuge verändern.
