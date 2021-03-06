# Lab 12: Eigene Templates erstellen

Im Unterschied zum [Lab 11](11_template.md) schreiben/definieren wir hier unsere eigenen Templates bevor wir damit Applikationen erstellen.

## Hilfreiche oc client Befehle
Auflisten aller Befehle:
```
$ oc help
```

Konzepte und Typen:
```
$ oc types
```

Übersicht aller Resourcen:
```
$ oc get all
```

Infos zu einer Resource:
```
$ oc get <RESOURCE_TYPE> <RESOURCE_NAME>
$ oc describe <RESOURCE_TYPE> <RESOURCE_NAME>
```

## Generierung
Über "oc new-app" oder "Add to project" in der Web Konsole werden die Resourcen automatisch angelegt. In der Web Konsole kann die Erstellung einfach Konfiguriert werden.

Für den produktiven Einsatz reicht das meinstens nicht aus. Da braucht es mehr Kontrolle über die Konfiguration. Dazu sind eigene Templates die Lösung. Sie müssen jedoch nicht von Hand geschrieben werden sondern können als Vorlage generiert werden.

### Generierung vor Erstellung
Mit **oc new-app** parst OpenShift die gegebenen Images, Templates, Source Code Repositories usw. und erstellt die Resourcen Definitionen. Mit der Option **-o** erhält man die Definitionen ohne dass die Resourcen angelegt werden.

So sieht die Definition vom hello-world Image aus.
```
$ oc new-app hello-world -o json
```

Spannend ist auch zu schauen, was OpenShift aus einem eigenen Projekt machen würde. Dafür ein Git Repository oder einen lokalen Pfad vom Rechner angeben.

Beispiel Befehl, wenn man sich im Root Ordner vom Projekt befindet:
```
$ oc new-app . -o json
```
Wenn verschiede Imagestreams in Frage kommen könnten oder keiner gefunden wurde, muss er angeben werden:
```
$ oc new-app . --image-stream=wildfly:latest -o json
```

`oc new-app` erstellt immer eine Liste von Ressourcen. Bei Bedarf kann eine solche mit [jq](https://stedolan.github.io/jq/) in ein Template konvertiert werden:
```
$ oc new-app . --image-stream=wildfly:latest -o json | \
  jq '{ kind: "Template", apiVersion: .apiVersion, metadata: {name: "mytemplate" }, objects: .items }'
```

### Generierung nach Erstellung
Bestehende Resourcen werden mit **oc export** exportiert.
```
$ oc export route my-route
```

Welche Resourcen braucht es?

Für ein vollständiges Template sind folgende Resourcen notwendig:
* Image Streams
* Build Configurations
* Deployment Configurations
* Persistent-Volume Claims
* Routes
* Services

Beispiel Befehl für einen Export der wichtigsten Resourcen als Template.
```
$ oc export is,bc,pvc,dc,route,service --as-template=my-template -o json > my-template.json
```
Ohne die Option *--as-template* würde eine Liste von Items anstatt einem Template mit Objects exportiert.

Momentan gibt es ein offenes [issue](https://github.com/openshift/origin/issues/8327) welches zur Folge hat,
dass image streams nach dem re-importieren nicht mehr korrekt funktionieren. Als Workaround kann das Attribut
`.spec.dockerImageRepository` wo vorhanden mit dem Wert des Attributes `.tags[0].annotations["openshift.io/imported-from"]`
ersetzt werden. Mit [jq](https://stedolan.github.io/jq/) kann dies gleich automatisch erledigt werden:

```
$ oc export is,bc,pvc,dc,route,service --as-template=my-template -o json |
  jq '(.objects[] | select(.kind == "ImageStream") | .spec) |= \
    (.dockerImageRepository = .tags[0].annotations["openshift.io/imported-from"])' > my-template.json
```

Attribute mit Wert `null` sowie die Annotation `openshift.io/generated-by` dürfen aus dem Template entfernt werden.

### Vorhandene Templates exportieren
Es können auch bestehenden Templates der Plattform abgeholt werden um eigene Templates zu erstellen.

Verfügbare Templates sind im OpenShift Namespace hinterlegt. So werden alle Templates aufgelistet:
```
$ oc get templates -n openshift
```

So erhalten wir eine Kopie vom eap70-mysql-persistent-s2i Template.
```
$ oc export template eap70-mysql-persistent-s2i -o json -n openshift > eap70-mysql-persistent-s2i.json
```

## Parameter
Damit die Applikationen für die eigenen Bedürfnisse angepasst werden kann, gibt es Parameter. Generierte oder exportierte Templates sollten fixe Werte, wie Hostnamen oder Passwörter, durch Parameter ersetzt werden.

### Parameter von Templates Anzeigen
Mit **oc process --parameters** werden die Parameter eines Templates angezeigt. Hier wollen wir sehen, welche Paramter im Cakephp MySQL Template definiert sind.
```
$ oc process --parameters cakephp-mysql-example -n openshift
NAME                           DESCRIPTION                                                                GENERATOR VALUE
NAME                           The name assigned to all of the frontend objects defined in this template.           cakephp-mysql-example
NAMESPACE                      The OpenShift Namespace where the ImageStream resides.                               openshift
MEMORY_LIMIT                   Maximum amount of memory the CakePHP container can use.                              512Mi
MEMORY_MYSQL_LIMIT             Maximum amount of memory the MySQL container can use.                                512Mi
...
```

### Parameter von Templates mit Werten Ersetzen
Für die Erzeugung der Applikationen können gewünschte Parameter mit Werten ersetzt werden. Dazu **oc process** verwenden:
```
oc process -f eap70-mysql-persistent-s2i.json \
  -v PARAM1=value1,PARAM2=value2 > processed-template.json
```
So werden Parameter vom Template mit den gegebenen Werten ersetzt und in eine neue Datei geschrieben. Diese Datei wird eine Liste von Resourcen/Items sein, welche mit **oc create** erstellt werden können.
```
oc create -f processed-template.json
```
Dies kann auch in einem Schritt gemacht werden:
```
oc process -f eap70-mysql-persistent-s2i.json \
  -v PARAM1=value1,PARAM2=value2 \
  | oc create -f -
```

## Templates schreiben
OpenShift Dokumentation:
* [Template Konzept](https://docs.openshift.com/container-platform/3.3/architecture/core_concepts/templates.html)
* [Templates Schreiben](https://docs.openshift.com/container-platform/3.3/dev_guide/templates.html)

Applikationen sollten so gebaut werden, dass sich pro Umgebung nur ein paar Konfigurationen unterscheiden. Diese Werte werden im Template als Parameter definiert.
Somit ist der erste Schritt nach dem Generieren einer Template Definition, das Definieren von Parameter. Das Template wird mit Variablen erweitert, welche dann mit den Parameter Werten ersetzt werden. So wird die Variable `${DB_PASSWORD}` mit dem Parameter mit dem Namen `DB_PASSWORD` ersetzt.

### Generierte Parameter
Oft werden Passwörter automatisch generiert, da der Wert nur im OpenShift Projekt verwendet wird. Dies kann mit einer Generate Definition erreicht werden.
```
parameters:
  - name: DB_PASSWORD
    description: "DB connection password"
    generate: expression
    from: "[a-zA-Z0-9]{13}"
```
Diese Definition würde ein zufälliges, 13 Zeichen langes Passwort mit Klein-, Grossbuchstaben und Zahlen generieren.

Auch wenn ein Parameter mit Generate Definition konfiguriert ist, kann er bei der Erzeugung überschrieben werden.

### Template Merge
Wenn z.B eine App mit einer Datenbank zusammen verwendet wird, können die zwei Templates zusammengelegt werden. Dabei ist es wichtig die Template Parameter zu konsolidieren. Dies sind meinstens Werte für die Verbindung zur Datenbank. Dabei einfach in beiden Templates die gleiche Variable vom gemeinsamen Parameter verwenden.

## Anwenden vom Templates
Templates können mit `oc new-app -f <FILE>|<URL> -p <PARAM1>=<VALUE1>,<PARAM2>=<VALUE2>...` instanziert werden.
Wenn die Parameter vom Template bereits mit `oc process` gesetzt wurden, braucht es die Angabe der Parameter nicht mehr.

### Metadata / Labels
`oc new-app` fügt standardmässig das Label `app=<TEMPLATE NAME>` in alle instanzierten Resourcen ein. Bei einigen
OpenShift Versionen kann dies zu [ungültigen](https://github.com/openshift/origin/issues/10782) Resourcendefinitionen führen.
Als Workaround kann mit `oc new-app -l <LABEL NAME>=<LABEL VALUE> ...` ein alternatives Label konfiguriert werden.

## Ressourcen aus docker-compose.yml erstellen

Seit Version 3.3 bietet die OpenShift Container Platform die Möglichkeit Resourcen aus der Docker Compose Konfigurationdatei `docker-compose.yml` zu erstellen. Diese Funktionalität ist noch als experimentell eingestuft. Beispiel:
```
git clone -b techlab https://github.com/appuio/weblate-docker#techlab
oc import docker-compose -f docker-compose.yml -o json
```

Die Möglichkeit eine Datei direkt via URL zu importieren ist vorgesehen aber noch nicht implementiert. Durch weglassen der Option `-o json` werden die Resourcen direkt angelegt statt ausgegeben. Momentan werden Services für bereits vorhandene Docker images nur angelegt falls eine explizite Portkonfiguration in `docker-compose.yml` vorhanden ist. Diese können in der Zwischenzeit mit Hilfe von `oc new-app` angelegt werden:
```
oc new-app --name=database postgres:9.4 -o json|jq '.items[] | select(.kind == "Service")' | oc create -f -
oc new-app --name=cache memcached:1.4 -o json|jq '.items[] | select(.kind == "Service")'|oc create -f -
```

---

**Ende Lab 12**

[<< zurück zur Übersicht] (../README.md)
