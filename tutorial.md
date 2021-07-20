# Step 1: GitHub Link kopieren 

wie immer sind alle Dateien hier im GitHub repo vorhanden. 

https://github.com/echtelerp/IoT-Stack-nodered-anpassung

lokaler node-red container zum testen: 
> docker run -it -p 1880:1880 -v node_red_data:/data --name mynodered nodered/node-red

Intro: 
in den letzten Videos haben wir 1) einen Pi mit docker ausgesattet, und 2) den typischen IoT stack zum laufen gebracht. 
wie im letzten Video schon erwähnt, machen wir den Aufwand ja mit einem bestimmten Grund: 
Wir wollen unsere flows, dashboards, etc. alle mit einem Befehl sarten. 

# Step 2: 
Heute schauen wir uns die Anpassung des Node Red containers an. 
dies kann von klein bis groß gehen. 

Relativ einfach anpassbar sind: 
1) settings.js datei
2) flows.json datei
3) bibliotheken einbinden
4) dashboard themes
Etwas komplexer wird es bei:
5) CSS datei - advanced :-)
und richtig Aufwändig aber dann maximal angepasst wird es bei 
6) Vorinsallierte nodes - High Advanced! 


# Step 3: Vorbereitungen - Verzeichnisse erstellen:
wir starten mit dem erstellen der Ordner und der Verzeichnis Struktur für die Anpassungen die wir vornehmen wollen:
zuerst hinzufügen des provisioning ordners
die mit > gekennzeichneten befehle sind am Pi, in VS code geht das manuell schneller :-)
> mkdir provisioning 
hinzufügen des darunterliegenden ordners... 
> cd provisioning
> mkdir assets
> mkdir config
> mkdir flows
> mkdir lib 
> mkdir styling

# Step 4: Dockerfile
jetzt geht es an die Änderungen im Dockerfile. 
NodeRed kann vor dem start mit der settings und der flows datei "gespeist" werden. 
Dies geschieht durch kopieren der Dateien in die vorgesehenen Ordner: 

Änderungen am Dockerfile: 

----unterhalb kopieren-----

#zusatzinfos hinzufügen
LABEL maintainer "dein Name"
LABEL Version "1.0"
LABEL TAG "NodeRed dein Name Style"

#copy assets into directory
COPY /provisioning/styling/newcss.css /usr/src/app/styling

#copy settings.js file to /data directory
COPY /provisioning/config/settings.js /data

#copy flow file from local dir to host to /data
COPY /provisioning/flows/flows.json /data

#copy flows to library
COPY /provisioning/lib/flows/ /data/lib/flows/

#copy theme Styling to Theme Library
COPY /provisioning/lib/themes/ /data/lib/themes

-----überhalb kopieren-------


# Step 5: settings.js datei
die Settings Datei konfiguriert node red.
Hier können einige änderungen gemacht werden. 
z.B. Port, http root und styling sowie passwort...
https://nodered.org/docs/user-guide/runtime/configuration

Fangen wir oben an: 
- Zeile 16 NodeRed port
- ab Zeile 130 - Passwortschutz
- ab 189 - Cors funktion
- Zeile 273 - Standard Sprache
- ab Zeile 336 - editor Theme 

## generieren des Passwortes: 

zuerst starten wir einen lokalen container um das PW zu erstellen: 
> Docker run -it -p 1880:1880 -v node_red_data:/data --name mynodered nodered/node-red
neue shell öffnen: 

> docker exec -d mynodered node -e "console.log(require('bcryptjs').hashSync(process.argv[1], 8));" hierpassworteintragen
wichtig am ende muss das neue Wunschpasswort stehen. 

das ergebnis sollte ungefähr so aussehen...: 
$2a$08$B6uYs4pJxtnSNX1x.TF.vO6WSn9Ft1NWX8ujnNHvrmDyg3QCLQ3BO 

dieses PW muss nun in der Settings Datei eingetragen werden. 

die nodered instanz mit 
> ctrl+C schließen
> docker container prune
> docker system df  ---> zeigt noch die übersicht

Alle änderungen in der Settings datei müssen durch entfernen der // am Anfang der Zeile "aktiviert" werden. 
passt dabei auf, dass die Javascript struktur eingehalten bleibt. Stichwort Kommas, klammern, etc. 

# Step 5: flows.json 

die flows.json bekommt ihr als download nachdem ihr beispielsweise einen flow gemacht habt, den ihr immer wieder haben wollt. 
das ist sinnvoll, wenn die Node-Red Instanz nach dem engineering in die Produktionsphase übergehen soll. 

Download der Datei 
-> ablegen unter config/...

# Step 6: Bibliothek vorbefüllen. 

node Red bietet seit der version 1.x die möglichkeit code schnipsel in der Lib abzulegen. 
beispielsweise häufig verwendete selbsgeschriebene Function Blocks. 

-> diese müssen in /lib/flows

# Step 7: theme - Dashboard stylings... 

wenn ihr euer Node-Red dashboard gestylet habt, könnt ihr diese konfiguration ebenfalls vorab laden lassen. 


# Step 8: CSS - advanced! 
Jetzt wird es knifflig. 
das macht man in der Regel nur einmal :-)

Zuerst das gesamte Repository auf den Rechner laden. 
https://github.com/node-red/node-red
jetzt müsst ihr node installiert haben.  
alternativ kann das aber auch innerhalb eines Docker Containers erfolgen. 
das werden wir im Verlauf nun machen: 

docker lokal oder remote: 
> docker run -it --name sasschange node:stretch bash  
> git clone https://github.com/node-red/node-red.git
> ls -la            um die Dateien anzuzeigen
> cd node-red
> npm install nopt
> npm install path
> npm install fs
> npm install node-sass
> mkdir changesass
> cp packages/node_modules/@node-red/editor-client/src/sass/colors.scss 
> cd changesass


nun benötigen wir ein neues bash fenster:

> docker cp sasschange:/node-red/changesass/colors.sass .

Jetzt öffen wir die Sass datei und machen unsere Änderungen. 

Die Änderungen müssen wir jetzt wieder zurück in den Container packen: 

> docker cp ./colors.sass sasschange:/node-red/scripts


jetzt wechseln wir in die shell die auf den container zeigt: 

> cd ..
> cd node-red
> cd scripts

> node build-custom-theme.js --in .\colors.scss --out newcss.css

jetzt müssen wir die newcss.css noch kopieren

> docker cp sasschange:/node-red/scripts/newcss.css .

die neue CSS Datei muss nun noch in den entsprechenden Ordner :-)
---> /provisioning/styling


# Step 9: Custom nodered mit vorinstallierten Modules: 

jetzt wird es maximal kompliziert! 

QUELLE: https://github.com/node-red/node-red-docker

wir benötigen das repo bei uns lokal. dazu clonen wir das repo
oder laden es per zip herunter. 
im Ordner nodered: 
> git clone https://github.com/node-red/node-red-docker.git

den Inhalt des docker-custom ordners packen wir in das nodered verzeichnis. 
den rest können wir löschen

jetzt passen wir die docker-compose.yaml datei an: 

    bei service nodered, packen wir folgende build struktur dazu: 

-----------

    build: 
      context: ./nodered
      dockerfile: Dockerfile.custom
      args: 
        ARCH: arm32v7
        NODE_VERSION: 12
        NODE_RED_VERSION: 1.3.5
        OS: alpine
        BUILD_DATE: 20.07.2021
        TAG_SUFFIX: default

------------

dabei ändern wir die version, die arch, und das date manuell. 


als nächstes schauen wir die package.json datei an: 
dort können wir unter dependencies weitere nodes defineiren. 
wir fügen testweise: 

        "node-red-dashboard":"*",
        "node-red-contrib-influxdb":"*",
        "node-red-contrib-telegrambot":"*"

hinzu. dadurch werden schon beim build die beiden nodes mit insatlliert. 


jetzt müssen noch die Kopier einstellungen im dockerfile ans ende hinzugefügt werden:

----------

#copy assets into directory
COPY /provisioning/styling/newcss.css /usr/src/app/styling

#copy settings.js file to /data directory
COPY /provisioning/config/settings.js /data

#copy flow file from local dir to host to /data
COPY /provisioning/flows/flows.json /data

#copy flows to library
COPY /provisioning/lib/flows/ /data/lib/flows/

#copy theme Styling to Theme Library
COPY /provisioning/lib/themes/ /data/lib/themes

------------------


# Step 10: alles zusammen auf den Pi bringen

dazu benötigen wir am besten FileZilla: 
https://filezilla-project.org

wir geben die IP oder den namen des Pi an
dockerPi 
dann das PW
dann noch den port 22 und wählen sftp als protokoll aus

schon sind wir auf dem Pi. 

# Abschließende Reinigungs-arbeiten: 

auf dem PC: 
> docker system df

dann löschen wir noch die test container und images mit 

> docker rmi imagename
> docker volume prune
> docker container prune



