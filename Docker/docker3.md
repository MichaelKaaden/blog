# Multi-Stage Builds oder: Immer die Build-Umgebung dabei haben

> Multi-Stage Builds für Angular: Wie Sie Apps gleich im Container bauen und
> trotzdem schlanke Docker-Images erstellen. Vermeiden Sie Risiken durch externe
> Abhängigkeiten!

Sie finden den Code zum Artikel auf
[GitHub](https://github.com/MichaelKaaden/dockerized-app/tree/master/Part-3-Multi-Stage-Build).

In Teil I dieser Artikelserie haben Sie gelernt, wie Sie Ihre Angular-App in ein
Docker-Image packen und in einem Container zur Ausführung bringen können.

In Teil II haben wir uns damit beschäftigt, die Konfiguration der Angular-App
mit Docker-Mitteln vorgeben zu können.

In diesem letzten Teil der Serie kümmern wir uns darum, die Build-Umgebung
deskriptiv in Ihr jeweiliges Projekt einzubinden und somit über die
Projektlaufzeit hinweg unter Ihrer Kontrolle zu halten.

---

## Interne und externe Abhängigkeiten

Kennen Sie das? Sie entwickeln ein Projekt nach allen Regeln der Ingenieurskunst
durch, übergeben es dem Betrieb oder dem Kunden -- und dann fassen Sie es so
lange nicht mehr an, bis sich jemand bei Ihnen meldet und nach Änderungen
verlangt.

Das ist dann der Moment, an dem Sie den Staub vom Projekt pusten und sich
vielleicht als erstes fragen, wie Sie das damals gebaut haben. Beschränken wir
uns auf TypeScript- oder JavaScript-Projekte, dann haben Sie vielleicht in der
`scripts`-Section im `package.json` das entsprechende Kommando hinterlegt.
Vielleicht müssen Sie auch ins `gulpfile.js` schauen.

Haben Sie sich besonnen und den Build angestoßen, stürzt Gulp unerwartet mit
einer C++-Exception ab, und Sie fangen an, auf Google nach der Ursache zu
suchen. Sie stellen dann fest, dass es wohl daran liegt, dass Sie derzeit mit
Node.js in Version 10 entwickeln, damals aber... Hm, war es Version 8 oder gar
noch Version 6? Und schon laden Sie beide Versionen herunter und probieren
herum. Schnell ist dann die erste Stunde investiert, ohne dass Sie produktiv
gewesen wären. Dabei ist Zeit doch Geld...

Was ist schiefgelaufen? Sie haben nicht beachtet, dass Sie es in Ihrem Projekt
nicht nur mit _internen_ Abhängigkeiten zu tun haben, die Sie fein säuberlich in
der `package.json`-Datei aufführen, sondern auch mit _externen_ Abhängigkeiten,
etwa der eingesetzten Node.js-Version.

## Externe Abhängigkeiten handhaben

Natürlich gibt es Lösungen, um mehrere Versionen von Node.js gleichzeitig auf
Ihrem Rechner vorzuhalten und zwischen diesen zu wechseln, etwa mit Tools wie
[n](https://github.com/tj/n) oder [nvm](https://github.com/creationix/nvm), aber
das ist gar nicht mein Punkt. Der Punkt ist, dass Sie _daran denken_ müssen,
jeweils auf die richtige Version umzustellen, wenn Sie zwischen Ihren Projekten
wechseln.

Auf Ihrem Build-Server, etwa [Jenkins](https://jenkins.io/), können Sie für
jedes Projekt eine individuelle Umgebung vorgeben. Projekt A baut dann mit
Node.js in Version 10, während das ältere Projekt B mit Node.js 6 erstellt
werden. Das ist eine tolle Sache, hilft Ihnen auf Ihrem Entwicklungsrechner aber
nicht weiter. Die Idee ist jedoch die Richtige: So etwas müssen wir dort
irgendwie auch erreichen.

## Docker To The Rescue

Warum bauen wir die App nicht einfach von einem Docker-Container aus? Dank des
Containers hätten wir feingranulare Kontrolle über die zu verwendende Node.js-
und npm-Version sowie das weitere Tooling, und all das könnten wir mittels eines
Dockerfiles in unserem Projekt ablegen und zusammen mit dem Projekt
versionieren.

Die Antwort ist ganz einfach: Das würde prima funktionieren, nur leider würde
das dabei entstehende Image sehr groß -- schließlich sind alle Tools und das
komplette `node_modules`-Verzeichnis Teil des Image, obwohl Sie diese nach dem
erfolgreichen Build nicht mehr brauchen. Alleine das `node_modules`-Verzeichnis
einer Angular-App ist schon fast 500MB groß. Zusammen mit Google Chrome für die
Unit Tests und Node.js sind Sie somit bei rund 1GB pro Image. Und Sie wissen ja:
Uns reicht ein nginx-Image mit der Kopie des `dist/<meine app>`-Verzeichnisses,
das lediglich rund 100MB groß wäre.

Damit ist unser Plan klar: Wir bauen die App im einen Container und erzeugen aus
dem Ergebnis ein neues, minimales Image. Zu unserem Glück ist das ein Problem,
das nicht nur uns beschäftigt, so dass Docker eine Lösung für genau diesen
Anwendungsfall bietet: Den sog. _Multi-Stage Build_.

## Multi-Stage Builds

Multi-Stage Builds kaskadieren den Build mehrerer Images und kopieren dabei
Daten vom Vorgänger in den Nachfolger. Lediglich das letzte Image ist dabei Ihr
Ergebnis, die Vorgänger spielen keine Rolle mehr (werden aber gecachet, um den
nächsten Build zu beschleunigen). Letzten Endes können Sie sich das wie bei
Prozessen und Pipes unter UNIX vorstellen: Die Ausgabe vom Vorgänger landet im
Nachfolger.

In unserem Fall muss das erste Image die App bauen, während das zweite die
erzeugte App aufnimmt. Daher muss das erste Image Node.js, npm, @angular/cli und
Google Chrome enthalten. Das zweite Image ist identisch mit dem, das wir in den
bisherigen Teilen der Artikelserie entwickelt haben, bezieht die App aber aus
dem ersten Image statt aus dem `dist`-Verzeichnis.

Soviel zum Plan, nun setzen wir das Ganze um.

Als erstes müssen wir unser `.dockerignore` bereinigen, denn nun legen wir sehr
wohl Wert auf alle Dateien und Verzeichnisse, die wir brauchen, um unsere App zu
erstellen. Andererseits benötigen wir jetzt das `dist`-Verzeichnis nicht mehr,
das zuvor noch essentiell für uns war.

```
.editorconfig
.git
.gitignore
.idea
README.md
coverage
dist
node_modules
```

Nun müssen wir die `src/karma.conf.js` anpassen, damit Google Chrome im Headless
Mode in einem Container unter Debian GNU/Linux funktioniert. Fügen Sie dazu dem
`config.set({...})`-Abschnitt folgendes hinzu:

```
customLaunchers: {
   ChromeHeadlessNoSandbox: {
       base: "ChromeHeadless",
       flags: ["--no-sandbox"],
   },
},
```

Hintergrund ist, dass wir das Sandboxing ausschalten müssen, damit die Tests
ausgeführt werden. Da Sie selbst den Container unter Kontrolle haben, sollte
dieses Risiko akzeptabel sein.

Als Nächstes müssen wir unser `Dockerfile` erweitern:

```dockerfile
FROM node:10-buster as node

RUN npm install -g @angular/cli@7.3.5

# install Google Chrome
RUN wget -q -O - https://dl-ssl.google.com/linux/linux_signing_key.pub | apt-key add - \
    && sh -c 'echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google.list' \
    && apt-get update && apt-get install -yq google-chrome-stable

# now test and build the app
WORKDIR /usr/src/app
COPY . ./
RUN yarn install
# ChromeHeadless needs to be run with --no-sandbox
RUN ng test --watch=false --browsers=ChromeHeadlessNoSandbox && ng build --prod

# Stage 2
FROM nginx

LABEL maintainer="Ihr Name <you@your.domain>"

COPY nginx/default.conf /etc/nginx/conf.d
COPY --from=node /usr/src/app/dist/dockerized-app /usr/share/nginx/html
```

Eine kurze Erklärung dazu: Dieses Dockerfile basiert auf einem Image mit Node.js
10 und legt darin `@angular/cli` in Version 7.3.5 (passend zur Version in der
`package.json`-Datei) und den Chrome-Browser ab. Anschließend baut es die App,
genau so, wie wir es bisher von Hand getan haben -- na, nicht ganz, wir lassen
nun die Tests laufen, denn das gehört doch sicher auch bei Ihnen dazu, nicht
wahr? Ansonsten kommentieren Sie die "RUN ng test ..."-Zeile einfach aus. Im
zweiten Schritt (gekennzeichnet durch den Kommentar "Stage 2") kopiert es die
fertiggestellte App aus dem ersten Image in das zweite.

Die entscheidenden Stellen sind `FROM node:10-buster as node`, die die
Bezeichnung `node` für das erste Image vorgibt, und `COPY --from=node ...`, die
unter Verwendung dieser Bezeichnung aus dem ersten in das zweite Image kopiert.

Jetzt entfernen wir noch den Build der App aus dem `dockerize.sh`-Skript, da
sich das `Dockerfile` ab jetzt um diesen Schritt kümmert:

```bash
#!/bin/bash
docker build -t dockerized-app .
```

## Bau des Images

Um das Image mit der App zu bauen, gehen wir genauso vor wie bisher: Wir führen
erst `dockerize.sh` und dann `redeploy.sh` aus. Hier ein Beispiellauf:

```console
$ ./dockerize.sh
Sending build context to Docker daemon  370.7kB
Step 1/11 : FROM node:10-buster as node
 ---> d71f0dc8e93b
Step 2/11 : RUN npm install -g @angular/cli@7.3.5
 ---> Running in 0ddecfd06f16
+ @angular/cli@7.3.5
added 289 packages from 181 contributors in 9.717s
Removing intermediate container 0ddecfd06f16
 ---> fd9b8afe37ec
Step 3/11 : RUN wget -q -O - https://dl-ssl.google.com/linux/linux_signing_key.pub | apt-key add -     && sh -c 'echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google.list'     && apt-get update && apt-get install -yq google-chrome-stable
 ---> Running in f85a552bb159
Get:1 http://deb.debian.org/debian buster InRelease [122 kB]
...
Fetched 8230 kB in 2s (5416 kB/s)
Reading package lists...
Reading package lists...
Building dependency tree...
Reading state information...
The following additional packages will be installed:
  adwaita-icon-theme at-spi2-core dbus dbus-user-session
  ...
Setting up google-chrome-stable (78.0.3904.97-1) ...
Removing intermediate container f85a552bb159
 ---> 92e5bb2bbaed
Step 4/11 : WORKDIR /usr/src/app
 ---> Running in 86a7a6c9204f
Removing intermediate container 86a7a6c9204f
 ---> 0c06bb302e39
Step 5/11 : COPY . ./
 ---> 9d6b58feea14
Step 6/11 : RUN yarn
 ---> Running in a35c44a28667
yarn install v1.19.1
[1/4] Resolving packages...
[2/4] Fetching packages...
[3/4] Linking dependencies...
[4/4] Building fresh packages...
Done in 17.33s.
Removing intermediate container a35c44a28667
 ---> 5959d3240af1
Step 7/11 : RUN ng test --watch=false --browsers=ChromeHeadlessNoSandbox && ng build --prod
 ---> Running in e8e73909da63
11 11 2019 17:51:51.637:INFO [karma-server]: Karma v4.0.1 server started at http://0.0.0.0:9876/
11 11 2019 17:51:51.638:INFO [launcher]: Launching browsers ChromeHeadlessNoSandbox with concurrency unlimited
11 11 2019 17:51:51.640:INFO [launcher]: Starting browser ChromeHeadless
11 11 2019 17:51:53.962:INFO [HeadlessChrome 78.0.3904 (Linux 0.0.0)]: Connected on socket x67ZoJs6ERhBn45OAAAA with id 98052412
TOTAL: 9 SUCCESS
TOTAL: 9 SUCCESS

Date: 2019-11-11T17:52:15.005Z
Hash: 046f0c97454b9144a096
Time: 17593ms
chunk {0} runtime.a5dd35324ddfd942bef1.js (runtime) 1.41 kB [entry] [rendered]
chunk {1} es2015-polyfills.4a4cfea0ce682043f4e9.js (es2015-polyfills) 56.4 kB [initial] [rendered]
chunk {2} main.0c51b538c84777d5bf5e.js (main) 262 kB [initial] [rendered]
chunk {3} polyfills.9f3702a215d30daac9b6.js (polyfills) 41 kB [initial] [rendered]
chunk {4} styles.3ff695c00d717f2d2a11.css (styles) 0 bytes [initial] [rendered]
Removing intermediate container e8e73909da63
 ---> 3a3d2d063b02
Step 8/11 : FROM nginx
 ---> 53f3fd8007f7
Step 9/11 : LABEL maintainer="Michael Kaaden <github@kaaden.net>"
 ---> Using cache
 ---> 974a15c23b4f
Step 10/11 : COPY nginx/default.conf /etc/nginx/conf.d
 ---> Using cache
 ---> cb36b6c88fbc
Step 11/11 : COPY --from=node /usr/src/app/dist/dockerized-app /usr/share/nginx/html
 ---> Using cache
 ---> 005ab4ca56a3
Successfully built 005ab4ca56a3
Successfully tagged dockerized-app-multistage:latest
```

Wenn Sie das Meldungspaar "Successfully built .../Successfully tagged ..."
sehen, haben Sie es geschafft: Der Multi-Stage Build hat geklappt. Führen Sie
Ihren Container nun aus:

```console
$ ./redeploy.sh
Removing network dockerized-app_default
WARNING: Network dockerized-app_default not found.
Creating network "dockerized-app_default" with the default driver
Creating dockerized-app_web_1 ... done
```

Der so erzeugte Container sollte nun genauso funktionieren wie der aus dem
vorigen Teil dieser Artikelserie. Die Verbesserung besteht jetzt darin, dass Sie
alles, was Sie zum Build benötigen, in Ihrem Projekt beschreiben und das in
Ihrer Quellcodeverwaltung mit versionieren können. Sie haben dadurch keinen
Performance-Nachteil, da Docker den ersten Stage cachet -- lediglich der zweite
Stage muss jedes Mal erstellt werden, wenn Sie etwas an Ihrer App ändern.

## Vergleich

Hier ein Größenvergleich der Images, die wir in jedem der drei Teile der
Artikelserie erstellt haben.

| REPOSITORY                | TAG    | IMAGE ID     | CREATED            | SIZE  |
| ------------------------- | ------ | ------------ | ------------------ | ----- |
| dockerized-app-simple     | latest | a90b35651f39 | 18 minutes ago     | 110MB |
| dockerized-app-env        | latest | 709da311ce4b | 17 minutes ago     | 110MB |
| dockerized-app-multistage | latest | 3ecfc4231dd5 | About a minute ago | 110MB |

Wie Sie sehen, gibt es keinen spürbaren Unterschied zwischen den Image-Größen.
Das ist natürlich kein Zufall: In den ersten beiden Teilen haben wir die App von
Hand gebaut und das `dist/dockerized-app` in das Image kopiert. In diesem
Blogbeitrag haben wir den Build in ein Image im ersten Stage verlagert und von
da aus kopiert. Es war also zu erwarten, dass sich die Größe des finalen Images
nicht ändert.

Spaßeshalber habe ich die Größe des ersten Stage des Multi-Stage Builds
gemessen:

| REPOSITORY | TAG    | IMAGE ID     | CREATED        | SIZE  |
| ---------- | ------ | ------------ | -------------- | ----- |
| stage1     | latest | 19e76e412adc | 25 seconds ago | 989MB |

1GB ist eine stolze Größe. Würden wir unsere App mit diesem Image betreiben,
hätten wir den zehnfachen Speicherbedarf...

## Grenzen der vorgestellten Lösung

Mit der vorgestellten Lösung können wir unsere App jederzeit mit den von uns
festgelegten NPM-Paketen bauen, zumindest unter der Annahme, dass diese Pakete
auch in Zukunft noch verfügbar sind. Die NPM Registry vergisst nichts, insofern
bin ich da äußerst zuversichtlich.

Mittelfristig verändern sich die Images natürlich, die die Basis der Lösung
darstellen. Node 10 wird in neueren Versionen vorliegen, das Debian 10-Image
wird ebenfalls mit Updates versorgt. Unsere App wird davon weitestgehend
unbeeinflusst bleiben. Allerdings benötigen einige NPM-Pakete wie `node-gyp`
beispielsweise sowohl den installierten Python-Interpreter als auch den
C++-Compiler. Das kann im Einzelfall zu der einen oder anderen Änderungen in der
von `ng build` erzeugten App führen, was meist nicht auffallen wird, weil Sie
sowieso das eine oder andere Sicherheitsupdate für von Ihnen verwendete
NPM-Pakete einpflegen müssen.

Betrachten wir einen Zeitraum von zehn Jahren, sieht die Situation schon weniger
rosig aus, weil es dann evtl. gar kein Node 10-Image mehr gibt...

Damit sollte klar sein, dass die vorgestellte Lösung keine Art von
Langzeit-Archivierung der Build-Umgebung bieten kann, weil kein Langzeit-Archiv
der Abhängigkeiten wie der Basis-Images existiert. Falls dennoch genau das für
Ihren Auftraggeber wichtig sein sollte, dann hat er das Problem typischerweise
schon selbst für seine eigene Software im Griff, so dass Sie auf dessen
Problemlösung zur Archivierung der Build-Umgebung zurückgreifen können (und
sollten). Falls Sie sich selber darum kümmern müssten, müssen Sie letztendlich
auf irgendeine Art für die Langzeitarchivierung der Abhängigkeiten aus obigem
`Dockerfile` sorgen.

Sie sehen, zumindest kurz- und mittelfristig brauchen Sie sich keine ernsthaften
Gedanken um Ihre Build-Umgebung zu machen. Langfristig sieht das allerdings
anders aus.

## Fazit

Welche Methode sollten Sie also für Ihren Anwendungsfall wählen? Die
Entscheidung ist meiner Meinung nach einfach. Wenn Sie mit den
`environment.ts`-Dateien auskommen, bleiben Sie bei der Lösung aus Teil I, um
Ihre App in einen Container zu packen. Möchten Sie die App in mehreren
Umgebungen betreiben und deswegen die Konfiguration ändern, landen Sie
automatisch bei der Lösung aus Teil II. Und wenn es Ihnen wichtig ist, Ihre
Build-Umgebung im Griff zu haben, verwenden Sie den Multi-Stage Build aus diesem
Artikel, die Sie mit jeder der beiden zuvor genannten Methoden kombinieren
können.

Wie auch immer Sie sich entscheiden: Sie werden schnell merken, wie angenehm es
ist, Ihre App mit Docker zum Laufen zu bringen.

Und falls Sie sich doch einmal dafür entscheiden, einen Build Server wie Jenkins
zu installieren und passend einzurichten, beschränkt sich Ihre finale Tätigkeit
in jedem Projekt auf ein `git push`, woraufhin der Build Server die neueste
Version Ihres Projekts auscheckt, den Multi-Stage Build auslöst und das Ergebnis
auf dem Zielsystem zum Laufen bringt.

Somit können Sie sich in Zukunft auf Ihre Kernkompetenzen beschränken und das
tun, was Ihnen Spaß macht: Tolle Software bauen. Um den Build und das Deployment
kümmern sich Ihre Automatismen.
