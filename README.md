<span align="center">

# Plantwatch


</span>
Mit dieser Anleitung ist es Ihnen möglich eine grafische Übersicht über Ihre Pflanzen zu bekommen. Es werden sowohl die Temperatur und Feuchtigkeitsdaten, sowie die der Pflanze aktuell zur Verfügung stehende Lux Zahl angezeigt.


## 1. Hardware

- Raspberry Pi (zum Testen wurde ein Zero 2 W verwendet)<br>
- USB Adapter Micro-USB
- 16 GB microSD-Karte<br>
- Bluetooth Pflanzensensoren (je Pflanze 1 Sensor)<br>
    Von diversen Herstellern verfügbar. <br>
    z.B. ohne Wertung: WANFEI, Royal Gardineer oder VegTrug. Alle aufgeführten Sensoren sind baugleich.<br>
    Eine günstigere Beschaffungsquelle stellen diverse Anbieter aus Fernost dar.


## 2. Installation des Pi

Damit der Rasberry Pi zum ersten Mal starten kann, muss zunächst die microSD Karte mit dem entsprechenden Betriebssysem installiert werden.
Dazu wird die SD-Karte in einen PC eingesteckt und mit Hilfe der Software "Pi Installer" installiert.

1.	Download Pi Installer von der Website https://www.raspberrypi.com

2.	Pi Installer starten und unter **"Betriebssystem"** "Raspberry Pi OS Lite" auswählen.
    Für die Installation auf einem Pi Zero ist ein Betriebsystem wie Pi OS Lite einer Ubuntu Installation vorzuziehen.

3.	SD-Karte am Computer einstecken. Unter **"SD Karte"** die gewünschte SD Karte auswählen.

4.	Mit Strg + Shift + X bzw. Control + Shift + X den Advance Mode öffnen.

    Einstelllungen nach eigenem Bedarf anpassen

    **Hierbei gilt es folgendes zu beachten:**

    - ssh muss aktiviert werden. Passwort selbst wählen.
        
    -  Wifi muss zumindest bei den Pi Zero Geräten eingerichtet werden. 
        (Achtung Pi Zero hat nur Wifi 4)

5.	Durch Auswahl der Funkion "Schreiben" fortfahren. Meldung mit "Ja" bestätigen.

6.	Einlegen der SD Karte in den Raspberry Pi und einstecken der Stromversorgung. Der Pi startet nach dem Einstecken der Stromversorung automatisch.

7.	Verbindung mit ssh aufbauen. <br>
    Dazu Terminal öffnen und zum Verbindungsaufbau folgenden Befehl kopieren und anpassen.

    ```shell
    ssh pi@ip-adresse_des_Pi
    ```

	Bsp.: ssh pi@192.168.15.2

    Das benötigte Passwort wurde hierfür unter Punkt 4 selbst gewählt.


## 3. Installation von Docker inkl. Docker Compose

1.  Installation von Docker
    Die Installation von Docker unter Rasperberry Pi OS Lite ist equivalent zu der von ubuntu.
    https://docs.docker.com/engine/install/ubuntu/


2.  Installation von Docker-Compose
    https://docs.docker.com/compose/install/  


## 4. Installation von Go

Go wird benötigt, um den flowercare-exporter verwenden zu können. Da dieser mit Hilfe der Programmiersprache Go entwickelt wurde.

```shell
sudo apt-get install golang
```
Hinweis: Während der Installation ist eine Bestätigung notwendig, diese mit “Y“ bestätigen.
Ein ggf. angezeigter Hinweis zum Kernel kann ignoriert werden.


## 5. Installation der Bluetooth Komponenten und des flowercare-exporter

Zum Auslesen der Sensoren Metric wird der in Go programmierte flowercare_exporter verwendet. Dieser ist sehr einfach aufgebaut im Vergleich zu einigen anderen exportern jüngeren Datums, was eine möglichst lange Kompatibilität mit den Sensoren sicherstellen soll. <br>
https://github.com/xperimental/flowercare-exporter

1.	Installation der Bluetooth-Komponenten
    Quelle: https://howchoo.com/pi/bluetooth-raspberry-pi

    ```shell
    sudo apt-get update
    sudo apt-get upgrade
    sudo apt-get install bluetooth bluez blueman
    ```
    Hinweis: Während der Installation ist eine Bestätigung notwendig, diese mit “Y“ bestätigen.
    
    Restart + erneutes Einloggen erforderlich (Der erneute Login kann etwas dauern).


2.	Bluetooth Test
    Funktionstest: Die Ausgabe sollten diverse Bluetooth Adressen sein.

    ```shell
    sudo hcitool lescan
    ```
    Beenden durch Strg + C bzw. Ctrl + C.

3.	Installation von flowercare-exporter

    Download der benötigten Installationsdateien, sowie die Installation in ein neues Verzeichnis "flower-exporter"

    ```shell
    git clone https://github.com/xperimental/flowercare-exporter.git
    cd flowercare-exporter
    go build .
    cd


## 6. Konfigurationsdateien für Docker, Prometheus, Grafana und flowercare-exporter anlegen

1.	Eigenen Ordner für die Docker Images und Dokumente anlegen

    Anlegen eines Ordners „sensor“ im Homeverzeichnis und Wechsel in dieses Verzeichnis.

    ```shell
    sudo mkdir sensor
    cd sensor/
    ```
    
2. Docker compose Dokument anlegen

    ```shell
    sudo nano docker-compose.yml
    ```

    In die Datei einfügen:
    ```shell
    version: "3.7"

    volumes:
      prometheus_data: {}
      grafana_data: {}

    services:
      prometheus:
        image: prom/prometheus
        volumes:
          - ./prometheus/:/etc/prometheus/
          - prometheus_data:/prometheus
        command:
          - '--config.file=/etc/prometheus/prometheus.yml'
          - '--storage.tsdb.path=/prometheus'
          - '--web.console.libraries=/usr/share/prometheus/console_libraries'
          - '--web.console.templates=/usr/share/prometheus/consoles'
        ports:
          - 9090:9090
        networks:
          - Sensor
        restart: always

      grafana:
        image: grafana/grafana
        depends_on:
          - prometheus
        ports:
          - 3000:3000
        volumes:
          - grafana_data:/var/lib/grafana
          - ./grafana/provisioning/:/etc/grafana/provisioning/
        networks:
          - Sensor
        restart: always

    networks:
      Sensor:
        driver: bridge
    ```
    Beenden mit Strg + X bzw. Ctrl + X, Y, Enter

3.	Prometheus config Dokument anlegen

    ```shell
    sudo mkdir prometheus
    sudo nano prometheus/prometheus.yml
    ```

    In die Datei einfügen und in die letzte Zeile die IP Adress des Pi eintragen:
    ```shell
    # my global config
    global:
      scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
      evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
      external_labels:
        monitor: 'codelab-monitor'

    # Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
    rule_files:
      # - "first.rules"
      # - "second.rules"

    # A scrape configuration containing exactly one endpoint to scrape:
    # Here it's Prometheus itself.
    scrape_configs:
      # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
      - job_name: 'prometheus'
          static_configs:
            - targets: ['localhost:9090']

      - job_name: 'flowercare'
        metrics_path: /metrics
        static_configs:
          - targets: ['192.168.15.2:9294']
    ```

4.	Bluetooth-Geräte scannen

    ```shell
    sudo hcitool lescan
    ```

	Die Flowercare Geräte sind leicht herauszufiltern. Bsp.:
    C4:7C:8D:AA:BB:CC Flower care

    Die gefundenen Geräte notieren.

5.	Flowercare-exporter einrichten

    Anlegen einer Service Datei

    ```shell
    sudo nano /etc/systemd/system/flowercare.service
    ```

    Einfügen und unter Environment die Geräte gem. Beispiel anpassen:

    ```shell
    Description=FlowerCareService

    Wants=network.target
    After=syslog.target network-online.target

    [Service]
    Type=simple
    Environment="s01=-s Tomaten =c4:7c:8d:aa:bb:c0"
    #Environment="s02=-s Erdbeeren=c4:7c:8d:aa:bb:c1"
    #Environment="s03=-s Himbeeren=c4:7c:8d:aa:bb:c2"
    #Environment="s04=-s Paprika=c4:7c:8d:aa:bb:c3"
    #Environment="s05=-s Palme_Links=c4:7c:8d:aa:bb:c4"
    #Environment="s06=-s Palme_Rechts=c4:7c:8d:aa:bb:c5"

    #hier die Environments mit $... hinterienandern eintragen
    #ExecStart=sudo home/pi/flowercare-exporter/./flowercare-exporter $s01 $s02 $s03 $s04 $s05 $s06
    ExecStart=sudo home/pi/flowercare-exporter/./flowercare-exporter $s01
    #Restart=on-failure
    RestartSec=10
    KillMode=process

    [Install]
    WantedBy=multi-user.target
    ```

    Nach dem Erstellen und nach jeder Änderung, der zuvor erstellten Service-Datei, muss diese aktiviert werden:
    ```shell
    sudo systemctl daemon-reload
    sudo systemctl enable flowercare.service
    sudo systemctl restart flowercare
    ```
    ```shell
    systemctl status flowercare
    ```

    Der Service sollte nun aktiv sein, ansonsten noch einmal die config prüfen und die Änderung wie zuvor beschrieben aktivieren.

    Status kann mit Strg + C bzw. Crtl + C beendet werden.

## 8. Prometheus und Grafana einrichten

1.	Prometheus starten

    Mit diesem Befehl werden Prometheus und Grafana das erste Mal heruntergeladen und anhand der zuvor angelegten Konfigurationsdateien voreingestellt.

    ```shell
    docker-compose up -d
    ```

    Im Browser folgendes eingeben:

    IP-Adresse_des_Pi:9090

    Bsp:
    192.168.1.5:9090

    <img src="Screenshot/Screenshot_Targets.png" alt="Prometheus" height="300px">

    Die IP Adresse sollte die von Ihnen angegeben Adresse sein und der Status sollte grün sein.

2.	Grafana starten

    Im Browser folgendes eingeben:

    IP-Adresse_des_Pi:3000

    Logindaten: <br>
    Name: admin <br>
    Passwort: admin <br>

    Nach dem ersten Einloggen wird man gebeten das Passwort zu ändern.

    Nun befindet man sich bereits in Grafana. 

3. Erstellen eines ersten Dashboards

    Da Grafana viele Möglichkeiten zur Gestaltung des Dashboard anbietet und nicht jede einzelne aufgeführt werden kann, hier 2 Beispiele zur Gestaltung:
    Zunächst muss ein neues Dashboard angelegt werden. Dazu im linken Menü das Plus auswählen. Über ''Add an empty panel'' gelangt man in das Design Board von Grafana.

    **Variante A** - Anzeige der akutellen Temperatur und Feuchtigkeit der einzelnen Pflanzen.

    <img src="Screenshot/Screenshot_je_Pflanze.png" alt="Dashboard je Pflanze" height="300px"> <br>
    Beispiel Variante A

    Zunächst wechseln wir von Time series zu Gauge, dazu ein kurzer Klick auf Time Series und dann Gauge wählen.
    Unter Metrics Browser wird dann die entsprechende Metric gewählt. 

    Hier ein Beispiel für die Temperatur einer Pflanze:
    ```shell
    flowercare_temperature_celsius{name="Wohnzimmer_Calathea"}
    ```

    Generell können so über + Querry weitere Pflanzen hinzugefügt werden. Bei der Betrachtung der einzelnen Pflanzen gelten je Pflanze eventuell andere Idealwerte, von daher ist eine Erstellung eines Panels je Pflanze empfohlen.
    Nach der Auswahl der Metric kann auf der rechten Seite noch der Name der Pflanze, sowie die Idealwerte einer Pflanze eingestellt werden.
    Den Namen in das Feld Title eintragen.

    Die Idealwerte markiert man mit Hilfe des Threesholds.<br>
    Dazu benötigt man 3 Bereiche. Base und der erste Wert sind bereits angelegt, den Dritten erzeugt man über den Button +Add threeshold.<br>
    Die Farbe bei Base wird auf orange gestellt. <br>
    Der Wert oberhalb von Base wird auf den niedrigsten empfohlenen Temperaturwert der Pflanze eingestellt. Im Beispiel der Calathea 12 und die Farbe auf Grün. <br>
    Den obersten Wert ersetzen wir ensprechend durch den empfohlenen Maximalwert, hier 32 und stellen die Farbe rot ein.

    Da der Maximalwert der Ansicht aus den existierenden Werten der Metric automatisch eingestellt wird und nicht für alle Pflanzen gleiche Werte existieren, empfiehlt es sich, die Maximal anzeigbaren Werte einheitlich manuell festzulegen. Dazu muss, ebenfalls im rechten Menü, unter standard options der Wert bei Max festgelegt werden.

    Mit einem Klick auf Apply kommt man in das Dashboard zurück. 

    Ein Klick auf den zuvor festgelegten Titel öffnet ein Menü, in dem unter More... - Duplicate das Panel dupliziert werden kann. 
    Zum Ändern eines Panels auf den Titel des Panels klicken und dann Edit wählen.
    Diese kann dann durch Anpassung des Titels, der Metric und ggf. der Idealwerte angepasst werden. Über das Duplicat empfiehlt es sich auch das erste Panel mit Feuchtigkeit anzulegen. Hier muss dann allerdings auch die Metric geändert werden. 

    Ein Beispiel für die Feuchtigkeit einer Pflanze:
    ```shell
    flowercare_moisture_percent{name="Wohnzimmer_Calathea"}
    ```

    Über den Button Add panel können weitere beliebige Panels aufgenommen werden.

    Die einzelnen Panels können auf dem Dashboard frei verschoben werden.

    Über die Diskette kann das aktuelle Dashboard gespeichert werden.

    Über das Zahnrad kann der Name des Dashboards angepasst werden.
    
    Selbstverständlich kann im Anschluss der Einstellung ein weiteres Dashboard mit der Variante B erstellt werden.

    **Variante B** - Anzeige von Temperatur, Feuchtigkeit und Helligerkeit über die letzten 24 Stunden.

    <img src="Screenshot/Screenshot_je_Raum.png" alt="Dashboard je Raum" height="300px"> <br>
    Beispiel Variante B

    Zunächst wird ein Panel für die Temperatur eines Raumes angelegt. Unter Metrics Browser wird die entsprechende Metric gewählt. 

    Hier ein Beispiel für die Temperatur einer Pflanze:
    ```shell
    flowercare_temperature_celsius{name="Wohnzimmer_Calathea"}
    ```
    über den Button +Querry können weitere Pflanzen, nach dem Schema des Beispiels, zu diesem Raum hinzugefügt werden.

    Im rechten Menü wird unter Panel-Options der Titel festgelegt. 
    Unter Standardoptions wird Degress (°) gewählt und der Maximalwert manuell festgelegt. Dies dient dazu, dass später alle Räume die selben Maximalwerte haben.

    Über Apply wird das Panel dem Dashboard hinzugefügt.

    Der Klick auf den zuvor festgelegten Titel öffnet ein Menü, in dem unter More... - Duplicate das Panel dupliziert werden kann. 
    Zum Ändern eines Panels auf den Titel des Panels klicken und dann Edit wählen. 

    Hier wird nun die Metric angepasst.

    Aus flowercare_temperature_celsius{...} <br>
    wird flowercare_moisture_percent{...}

    Der Titel wird angepasst und aus Degrees wird Humidity (%H) und der Maximalwert wird auf 80 gesetzt.

    Im Anschluss wieder mit Apply das Fenster schließen.

    Optional kann noch der Lux Wert eingebunden werden. Dazu erneut ein Panel duplizieren.
    Die Metric zu flowercare_brightness_lux{...} ändern. Da die Lux stark schwankt, kann man hier auch einen größeren Intervall verwenden. Dieses passt man unter der Metricquelle im Feld min Step an. Empfehlung 30m für 30 min. <br>
    Auf der rechten Seite wird erneut der Titel angepasst, unter Axis wird von Linar auf Logarithmic base 10 umgestellt. Den Min Wert auf 1 stellen und den Max Wert auf 10.000. Im Anschluss mit Apply bestätigen.

    Das Dashboard kann durch verschieben, vergrößern und verkleinern der Panele, mit Hilfe des Mauszeigers, angepasst werden. Ein Beispeil dazu kann man im vorherigen Screenshot sehen.

    Selbstverständlich kann im Anschluss der Einstellung ein weiteres Dashboard mit der Variante A erstellt werden.
    Auch ein Dahboard für die einzelen Räume mit Kombination der Variante A und B kann im Alltag praktikabel sein.
