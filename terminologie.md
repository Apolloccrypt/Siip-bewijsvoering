# Terminologie

De vaktermen en afkortingen uit deze bewijsvoering, in gewone taal. Elk item is
aanklikbaar vanuit de andere documenten.

## Identiteit en paspoort

<a id="bsn"></a>
### BSN
Het burgerservicenummer, je unieke nummer bij de Nederlandse overheid. Een private partij
mag het alleen verwerken met een specifieke wettelijke grondslag. Meer:
[rijksoverheid.nl](https://www.rijksoverheid.nl/onderwerpen/persoonsgegevens/burgerservicenummer-bsn).

<a id="mrz"></a>
### MRZ (machineleesbare zone)
De twee regels met tekens en chevrons (`<`) onderaan de fotopagina van een paspoort, en
in de chip als datagroep DG1. Bij een Nederlands paspoort van vóór 30 augustus 2021 staat
het BSN in het personal-number-veld van deze zone. Meer:
[Wikipedia](https://en.wikipedia.org/wiki/Machine-readable_passport).

<a id="td3"></a>
### TD3
Het formaat van een paspoort-MRZ: twee regels van elk 44 tekens. Elke positie in die
regels heeft een vaste betekenis (documentnummer, nationaliteit, geboortedatum, enzovoort).

<a id="datagroepen"></a>
### Datagroepen (DG1 t/m DG15)
De chip in een paspoort of ID-kaart bewaart gegevens in genummerde datagroepen. DG1 is de
machineleesbare zone, DG2 is de gezichtsfoto. Andere datagroepen bevatten aanvullende
document- en persoonsgegevens. DG3 (vingerafdruk) en DG4 (iris) zaten niet in deze app.

<a id="sod"></a>
### SOD (Document Security Object)
Het door de staat ondertekende object in de chip waarmee de echtheid van het document
passief te controleren is. Een soort digitale zegel op je paspoort. Meer:
[Wikipedia](https://en.wikipedia.org/wiki/Biometric_passport).

<a id="jpeg2000"></a>
### JPEG2000
Het beeldformaat waarin de gezichtsfoto (DG2) in de paspoortchip is opgeslagen. Scherper
dan de gedrukte foto op de pagina.

<a id="elfproef"></a>
### Elfproef
Een rekenkundige controle waarmee een geldig BSN te herkennen is. Een negencijferig getal
dat de elfproef doorstaat, is vrijwel zeker een echt BSN en geen toevallige reeks. Meer:
[Wikipedia](https://nl.wikipedia.org/wiki/Elfproef).

## Meten en versleuteling

<a id="certificate-pinning"></a>
### Certificate pinning
Een app die pint, vertrouwt alleen het echte servercertificaat en weigert elke tussenpartij.
Dat maakt gewoon meeluisteren met een proxy onmogelijk.

<a id="differentiele-pinning"></a>
### Differentiële pinning
Niet elk kanaal is even zwaar beveiligd. Bij deze apps zijn de gevoelige hosts (identiteit,
auth, tickets) wél gepind en is het contentkanaal dat niet.

<a id="mitm"></a>
### Man-in-the-middle (MITM)
Een meettechniek waarbij je verkeer via een tussenproxy leidt om het te lezen. Werkt niet
tegen certificate pinning. In dit onderzoek is die techniek niet gebruikt.

<a id="in-process-tap"></a>
### In-process tap (SSL_write / SSL_read)
De gebruikte methode. In een eigen kopie van de app, op een eigen toestel, worden de twee
functies gehaakt waarmee de app versleutelt en ontsleutelt (`SSL_write` en `SSL_read`). De
app verbindt normaal met de echte server; de gegevens zijn vlak voor versleuteling leesbaar
in het geheugen van de app. Geen tussenpartij, geen gebroken pinning.

<a id="libssl"></a>
### libssl / BoringSSL
De TLS-bibliotheek waarin `SSL_write` en `SSL_read` zitten. TLS is de versleuteling achter
het slotje in de browser.

<a id="okhttp"></a>
### okhttp / Retrofit
De netwerkbibliotheken waarmee de Android-laag van de app zijn verzoeken doet. De gevoelige
identiteitshosts lopen hierover; dat verklaart de differentiële pinning.

<a id="http2"></a>
### HTTP/2 en multipart/form-data
HTTP/2 is de moderne versie van het webprotocol. `multipart/form-data` is de manier waarop
meerdere benoemde bestanden (zoals `photoFile` en `dataGroup1`) in één verzoek worden
meegestuurd.

## Bewijsintegriteit

<a id="hash"></a>
### Hash / SHA-256
Een cryptografische vingerafdruk van een bestand. Verandert er één byte, dan verandert de
hash volledig. SHA-256 is de gebruikte variant. Zo is te controleren dat twee kopieën van
een bestand identiek zijn.

<a id="ots"></a>
### OpenTimestamps (.ots)
Een methode om onafhankelijk vast te leggen wanneer een bestand bestond, door de hash te
verankeren in de Bitcoin-blockchain. Een `.ots`-bestand is het bewijs daarvan. Zo staat de
meetdatum vast en is achteraf niet ongemerkt te wijzigen. Meer:
[opentimestamps.org](https://opentimestamps.org/).

## App-analyse

<a id="static"></a>
### Statische analyse (apktool, strings, dex, Dart AOT)
De app uit elkaar halen zonder hem te draaien. `apktool` pakt de app uit; `strings` leest de
leesbare tekst uit de binaries. `classes.dex` is de Android-code (Kotlin); `libapp.so` is de
Flutter-code (Dart AOT, vooraf gecompileerd).

<a id="il2cpp"></a>
### il2cpp / UnityTls
Bij de PSV-app (een Unity-app) loopt het netwerkverkeer via Unity's eigen code (il2cpp) en
eigen TLS-laag (UnityTls), buiten de systeem-`libssl`. Daardoor werkt de tap die bij de
andere apps werkt hier niet zomaar.

## Infrastructuur en jurisdictie

<a id="cname"></a>
### CNAME
Een DNS-alias: de ene naam wijst door naar de andere. `onboarding.siip.io` wijst via een
CNAME door naar een Amazon-adres, wat laat zien waar de server werkelijk staat.

<a id="elb"></a>
### ELB (Elastic Load Balancer) / AWS eu-west-1
Een verdeelpunt voor verkeer binnen de cloud van Amazon (AWS). `eu-west-1` is de
AWS-regio Ierland. De data staat fysiek in de EU, bij een Amerikaanse aanbieder.

<a id="cloud-act"></a>
### CLOUD Act
Een Amerikaanse wet die de VS het recht geeft een Amerikaans bedrijf te dwingen gegevens af
te geven, ook als die op servers in Europa staan. Meer:
[Wikipedia](https://nl.wikipedia.org/wiki/CLOUD_Act).

## Tracking en toestemming

<a id="telemetrie"></a>
### Telemetrie
Gebruiks- en foutinformatie die een app automatisch naar de maker of naar derden stuurt.

<a id="firebase"></a>
### Firebase / FID / GA4 / Crashlytics
Meetdiensten van Google. FID (Firebase Installation ID) is een vast nummer dat je installatie
herkenbaar maakt. GA4 is Google Analytics 4. Crashlytics verzamelt crashmeldingen.

<a id="sentry"></a>
### Sentry / DSN
Een Amerikaanse dienst voor foutrapportage. De DSN is het adres waar een app zijn meldingen
heen stuurt; aan een gedeelde DSN is te zien dat meerdere apps naar hetzelfde account melden.

<a id="consent-mode"></a>
### Consent Mode v2
De standaard van Google om toestemmingssignalen mee te sturen. Ontbreken die signalen, dan
draait de meting op de standaardstand zonder dat toestemming is doorgegeven.

<a id="cmp"></a>
### CMP / IAB TCF / Didomi
Een CMP is een toestemmingsplatform (de cookiebanner-machinerie). IAB TCF is de branchestandaard
daarvoor. Didomi is een van de leveranciers. PEC had geen CMP; PSV wel.

<a id="ad-id"></a>
### AD_ID (advertentie-ID)
Het reclamenummer van je telefoon waarmee adverteerders je herkennen.

<a id="openroaming"></a>
### OpenRoaming / DNA Spaces / Passpoint
Cisco-technologie voor automatisch verbinden met wifi (Passpoint / Hotspot 2.0) en voor
locatie- en aanwezigheidsanalyse in een gebouw (DNA Spaces). In de PSV-app aan je identiteit
gekoppeld.

## Juridisch kader

<a id="avg"></a>
### AVG
De Algemene verordening gegevensbescherming, de Europese privacywet. Meer:
[Wikipedia](https://nl.wikipedia.org/wiki/Algemene_verordening_gegevensbescherming).

<a id="dpia"></a>
### DPIA
Een gegevensbeschermingseffectbeoordeling: de verplichte toets die vooraf de privacyrisico's
van een verwerking in kaart brengt en afweegt.

<a id="telecomwet"></a>
### Telecommunicatiewet art. 11.7a
De Nederlandse cookiewet: volgtechnologie mag pas laden nadat je toestemming hebt gegeven.

<a id="white-label"></a>
### White-label
Eén kant-en-klaar sjabloon dat per klant met een ander logo wordt uitgeleverd. De apps van
PEC Zwolle, FC Eindhoven, ADO Den Haag en NAC Breda zijn zulke varianten van hetzelfde sjabloon.
