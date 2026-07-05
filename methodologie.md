# Methodologie

Twee sporen: statische analyse van de publieke app-binaries, en een dynamische meting
van het eigen netwerkverkeer. De twee bevestigen elkaar. Waar ze verschillen, is dat
in de documenten benoemd.

## Statische analyse

De app is uit elkaar gehaald met apktool en de symbolen zijn uit de binaries gelezen.

- **Dart-laag** (`libapp.so`, Dart AOT): het datamodel en de teksten. Hier staan het
  onboarding-model, de datagroep-velden en de aan de gebruiker getoonde consenttekst.
- **Kotlin-laag** (`classes.dex`): de NFC-uitlezing en het versturen. Hier staan de
  hardcoded endpoints en de netwerkbibliotheek (okhttp).

Voorbeeld, het onboarding-datamodel dat de ruwe chip-datagroepen draagt:

```
$ strings -n4 libapp.so | grep -E 'datagroup_[0-9]+_data|IcaoCard|MrtdCard'
datagroup_1_data  datagroup_2_data  datagroup_5_data  datagroup_6_data  datagroup_7_data
datagroup_11_data datagroup_12_data datagroup_13_data datagroup_14_data datagroup_15_data
IcaoCard.fromJson
MrtdCard.fromJson
package:siip_flutter_bridge/src/models/onboarding/cards/icao_card.dart
```

De hardcoded backend-endpoints:

```
$ strings -n5 classes.dex | grep -E 'https://(onboarding|iam|user|ticketing|mail).siip.io'
https://iam.siip.io/
https://mail.siip.io/api/
https://onboarding.siip.io/api/
https://ticketing.siip.io/api/
https://user.siip.io/api/
```

De uitlezing en het versturen zitten in native Kotlin over okhttp:

```
$ strings -n5 classes.dex | grep -E 'nfc.mrtd|dataGroup.*bin|okhttp'
dataGroup1.bin  dataGroup11.bin  dataGroup12.bin
io.siip.packages.plugin.nfc.mrtd.cards.icaoCard.IcaoCard
io/siip/packages/plugin/connectors/externalOnboarding/ExternalOnboardingConnector
okhttp/4.12.0
```

## Dynamische meting: een tap binnen de app, geen man-in-the-middle

Een gewone tussenproxy werkt niet. De gevoelige hosts (`onboarding.siip.io`,
`iam.siip.io`, `ticketing.siip.io`) pinnen hun certificaat: de app vertrouwt alleen het
echte servercertificaat van Siip en weigert elke tussenpartij. Een klassieke
man-in-the-middle brak dan ook af met `certificate unknown`.

Daarom is er binnen de app zelf meegelezen. In een eigen kopie van de app, op een eigen
niet-geroot toestel, is een hook geplaatst op de twee functies waarmee elke Android-app
versleutelt en ontsleutelt: `SSL_write` en `SSL_read` in `libssl.so` (BoringSSL). De app
verbindt met de echte server, de TLS-handshake en de pinning slagen normaal, en vlak
voordat de bytes versleuteld worden verstuurd zijn ze in leesbare vorm in het geheugen
van de app te lezen.

Geen tussenpartij. Geen omzeild certificaat. Geen gebroken pinning. Er wordt alleen
gelezen wat de eigen app op het eigen toestel verstuurt, op het moment dat zij het
verstuurt.

### Waarom dit werkt waar een proxy faalt

De differentiële pinning verklaart het verschil. De gevoelige identiteits- en
tickethosts lopen over een native Kotlin-laag (okhttp 4.12.0). Het contentkanaal
(`scl.siip.io`, nieuws en fixtures) loopt over de Flutter-laag en is niet gepind.
Een generieke Flutter-ontpinner raakt de native laag niet, en laat de identiteits-upload
dus dicht. De tap op `libssl.so` zit onder allebei de lagen, in de TLS-bibliotheek zelf,
en ziet wat er werkelijk over de lijn gaat.

## Web-ticketlaag

De websites waarop kaarten worden gekocht zijn los gemeten met een browser-scan in drie
standen: zonder interactie, na weigeren en na accepteren van cookies. Zo is te zien welke
volgtechnologie laadt vóórdat er toestemming is gegeven.

## Reproductie

- Statisch: apktool op de publieke APK, `strings`/`grep` op `libapp.so` en `classes.dex`.
- Dynamisch: een frida-gadget in de eigen herverpakte app, een hook op `libssl.so`
  `SSL_write`/`SSL_read`, gefilterd op `siip.io`, tijdens een registratie met NFC-scan.
- Web: een headless browser-scan van de ticketdomeinen in de drie consent-standen.

De ruwe bronbestanden zijn verzegeld en gehasht; zie [`SHA256SUMS.txt`](SHA256SUMS.txt).
