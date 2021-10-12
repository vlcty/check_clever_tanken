# check_clever_tanken

check_clever_tanken ist ein nach den Nagios Developer Guidelines entwickeltes Script um die Preise für Kraft- und Hilfsstoffe von der Webseite http://www.clever-tanken.de zu extrahieren.

## Ein Wort der Warnung

Ich bin nun E-Auto Fahrer. Damit werde ich dieses Plugin nicht mehr weiterentwickeln.

## Funktionsweise

Das Script erwartet als Parameter die Tankstellen-ID. Diese findet man in der URL. Am Beispiel einer Tankstelle in meiner Umgebung:

http://www.clever-tanken.de/tankstelle_details/23266

Diese Beispieltankstelle hat die ID 23266, die ich nun an das Script übergeben kann:

```
./check_clever_tanken --station 23266
```

Das Script extrahiert alle gefundenen Kraft- und Hilfsstoffe von der Webseite. Die Ausgabe sieht wie folgt aus:

```
HEM Frankenstr. 11:

Diesel: 1.21 €/l
Super E10: 1.39 €/l
Super E5: 1.41 €/l
SuperPlus: 1.49 €/l
|HEM_Frankenstr_11_Diesel=1.21 HEM_Frankenstr_11_Super_E10=1.39 HEM_Frankenstr_11_Super_E5=1.41 HEM_Frankenstr_11_SuperPlus=1.49
```

Die Kenner erkennen den Performance String für die maschinelle Weiterverarbeitung.

## Warum HTML parsen, wenn es die MTS-K APIs gibt

Eine gute Frage! Das System der MTS-K und deren Sub-Anbieter sind toll und reichen für die meisten Anwender vollkommen aus. Eine erste Version dieses Plugins hat auch die (API von Tankerkönig)[https://creativecommons.tankerkoenig.de/] verwendet. Die MTS-K sammel aktuell nur die Daten von folgenden Kraftstoffen:

- Super E5 (95 Oktan, 5% Bioethanol)
- Super E10 (95 Oktan, 10% Bioethanol)
- Diesel

Es gibt aber noch mehr Kraft- und Hilfsstoffe:

- Super Plus (98 Oktan)
- Premium Spritsorten der Marken: Shell V-Power, Aral Ultimate, OMV Maxx Motion, etc.
- Truckdiesel
- E85 (104 Oktan, 85% Bioethanol, 15% Super)
- AdBlue
- Autogas

Gerade Super Plus wird doch bestimmt häufiger verkauft, deswegen verstehe ich nicht ganz, warum die Preise nicht über die MTS-K API verfügbar sind. Auf clever-tanken.de werden sowohl die MTS-K Preise sowie von Nutzern beigesteuerte Preise angezeigt.

Einen Nachteil hat die Geschichte: Ändert die Webseite ihre Struktur, dann muss das Plugin wieder angepasst werden. Ich hoffe, dass die Seite mir keine Steine in den Weg legt und ihr Design regelmäßig ändert oder mit Hilfe von JavaScript verschleiert.

## Alarmierung

Das Plugin kann "alarmieren", wenn ein Preis unter ein Limit fällt. Dazu können die Wunschpreise mit übergeben werden. Zum Beispiel möchte ich mich alarmieren lassen, wenn der Preis für Super Plus an der obigen Tankstelle unter 1.50 € fällt. Der entsprechende Aufruf sieht dann wie folgt aus:

```
./check_clever_tanken --station 23266 --alarm "SuperPlus: 1.50"
```

Wichtig ist, das der Sprit genau so heißt wie in der Ausgabe und das zwischen dem Doppelpunkt und dem Limit ein Leerzeichen ist. Die Option kann mehrmals verwendet werden.

Die Ausgabe verändert sich, sollte der Preis unter ein gewisses Limit fallen:

```
HEM Frankenstr. 11:

Diesel: 1.21 €/l
Super E10: 1.39 €/l
Super E5: 1.41 €/l
*SuperPlus: 1.49 €/l -> 1 ct günstiger als das Limit von 1.50 €/l*
|HEM_Frankenstr_11_Diesel=1.21 HEM_Frankenstr_11_Super_E10=1.39 HEM_Frankenstr_11_Super_E5=1.41 HEM_Frankenstr_11_SuperPlus=1.49
```

Der Exit Code vom Plugin ist dann 1 anstelle von 0. Das führt dazu, das der Service im Nagios/Icinga gelb wird und eine Alarmierung stattfinden sollte.

## Integration in Icinga 2:

Ein entsprechendes CheckCommand für Icinga 2 liegt in der Datei "icinga2_CheckCommand.conf".

Ich habe dann auf meinen Monitoring Master folgendes Array in der Hostconfig angelegt:

```
   /*
    * Benzinpreise
    */
   vars.clever_tanken = [
       "50498",
       "26777",
       "26773",
       "23266",
       "104",
       "24109",
       "42298",
       "26160",
       "5883",
       "9849"
   ]
```

Dieses wird dann in einem Service benutzt:

```
apply Service "clever-tanken-" for ( stationID in host.vars.clever_tanken ) {
    import "generic-service"
    import "30minute-service"

    display_name = "Clever Tanken " + stationID
    command_endpoint = NodeName
    host_name = NodeName
    check_command = "clevertanken"

    vars.clevertanken_stationid = stationID
    vars.clevertanken_alarm = [
        "SuperPlus: 1.39"
    ]
}
```

Je nach Anwendungsgebiet kann man auch anstelle eines Arrays einen Hash verwenden. Dann hat man die Möglichkeit pro Tankstelle eigene Alarme zu definieren. In meinem Setup möchte ich alarmiert werden, sobald eine Tankstelle einen günstigeren Preis für Super Plus anbietet als 1,39 €.
