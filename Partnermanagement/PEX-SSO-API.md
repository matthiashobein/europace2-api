
PEX SSO (Silent Sign On) API
============================

EUROPACE 2 erlaubt es seinen Partnern, Benutzer per _Silent Sign On_ anzumelden. 

_Silent Sign On_ in ein Verfahren, das es ermöglicht, einen Benutzer von einer Webseite zu BaufiSmart weiter zu leiten, ohne das sich dieser erneut anmelden muss.

EUROPACE 2 nutzt dafür den Internet Standard [JWT](http://jwt.io) mit [RSA](http://de.wikipedia.org/wiki/RSA-Kryptosystem) als Signaturverfahren.

Dieses Dokument erläutert Ihnen die verwendeten technischen Standards und welche Schritte zu einer erfolgreichen Anbindung notwendig sind.

### Übersicht der notwendigen Schritte als Shell Skripte


Folgende Script Aufrufe spielen den Prozess exemplarisch durch. Benötigt werden der API-Key und die Partner-Id der Organisation (issuer) sowie die Partner-Id des einzuloggenden Benutzers.

Es muss Java 1.8 installiert sein.

```
./generate-key.sh
./upload-key.sh  <api-key> <partnerId_des_key_issuers>
./login-via-sso.sh <partnerId_des_einzuloggenden> <partnerId_des_key_issuers> [redirectTarget]
```

Erzeugung der RSA Keys
----------------------

```
openssl genrsa 2048 > rsa.private
openssl pkcs8 -topk8 -inform PEM -outform PEM -in rsa.private  -nocrypt > private-key.pem
openssl rsa -inform PEM -in rsa.private -pubout > public-key.pem
rm rsa.private
```

Hinterlegen des Public RSA Key in EUROPACE 2
--------------------------------------------

Der öffentliche RSA key muss in EUROPACE 2 hinterlegt sein, um die Signatur des JWT zu überprüfen. Der öffentliche Schlüssel kann per HTTP hochgeladen werden:

```
curl -X PUT
	 -H "Accept: application/json" 
	 -H "X-ApiKey: ${apiKey}" 
	 -H "X-PartnerId: ${partnerId}"
	 -H "X-TraceId: `date "+%Y-%m-%dT%H:%M:%S"`"
	 -H "Content-Type: text/plain;charset=utf-8"
	 --data @public-key.pem
	 https://www.europace2.de/partnermanagement/partner/${issuerId}/sso-pub-key
```

apiKey und partnerId müssen einer Organisationseinheit identifizieren, die "Einstellungsrechte" auf den _issuerId_ hat.
Die _issuerId_ entspricht der partnerId, für die der Key hinterlegt werden soll.

Erzeugen eines Json Web Tokens (JWT)
-------------------------------------

Für manuelle, explorative Tests der SSO Schnittstelle sowie als "Kopiervorlage" für Java basierte clients steht ein Java basiertes Testprojekt bereit.

Folgende Komponenten müssen installiert sein:

- [Java8](http://www.java.com/en/download/index.jsp)
- [Maven3](http://maven.apache.org/download.cgi)



1) Github Projekt auschecken:

```
git clone https://github.com/hypoport/jwt-toolbox.git
```

2) Projekt kompilieren und all-inclusive JAR erstellen.

```
cd jwt-toolbox
mvn package
```

3) Token erzeugen

```
java -jar target/jwt-toolbox-jar-with-dependencies.jar private-key.pem ${partnerToLogonId} ${issuerId}
```

Einloggen via JWT mit Redirect
------------------------------

Der soeben erzeugte token kann nun zur Anmeldung an EUROPACE 2 verwendet werden. 

Nun kann im Browser folgende URL aufgerufen werden:

```
https://www.europace2.de/partnermanagement/login?redirectTo=/uebersicht&authentication=${jwt}
```

Alternativ kann auch ein POST request mit gesetztem X-Authentication header verwenden werden:

```
POST /partnermanagement/login?redirectTo=/uebersicht&authentication=${jwt} HTTP/1.1
Host: www.europace2.de
X-Authentication: ${jwt}
X-TraceId: Eine_TraceId_zB_Datum_rueckwaerts
```

Response:

```
HTTP/1.1 302 FOUND
Location: /uebersicht
Set-Cookie: sessionId=dd800897698f8e2637d5c39e33083764 
```


### Schematischer Ablauf

Das folgende Sequenzdiagramm erläutert das Zusammenspiel der Kollaborateure in der HTTP Netzwerkebene:

![Sequenzdiagramm für die EUROPACE 2 Anmeldung via Silent Sign On](images/Sequence.png?raw=true)