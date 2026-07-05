# Technisch onderzoeksrapport: Siip / PDT

Datastroom- en identiteitsanalyse van de voetbal-app op basis van Persoonlijke Digitale
Toegang (PDT), leverancier Siip. Onderzoeksobject: de PEC Zwolle-app, met een vergelijking
naar de rest van de vloot en de PSV-app.

> Dit is geen penetratietest en geen beveiligingsonderzoek op andermans systeem. Het is een
> meting van het eigen netwerkverkeer op een eigen toestel, met een eigen paspoort. Er is
> geen systeem van Siip binnengedrongen en geen kwetsbaarheid misbruikt.

## Documentgegevens

| | |
|---|---|
| Onderzoeksobject | PEC Zwolle-app `io.siip.saas.pec` v2.1.1 (Flutter); vloot `io.siip.saas.<club>`; PSV-app `nl.psv` v8.6.0 |
| Backend | `*.siip.io` op AWS `eu-west-1` (Ierland); telemetrie naar Google en Sentry (VS) |
| Meetdatum | 1 en 2 juli 2026 |
| Toestel | OUKITEL WP19, Android 12, niet geroot; plus Android-14-emulator voor de Google-telemetrie |
| Methode | statische analyse van de publieke binaries + in-process TLS-tap op het eigen verkeer |
| Bewijs | verzegeld met SHA-256, hashlijst verankerd met OpenTimestamps (Bitcoin) |
| Status | eigen onderzoek; wederhoor van Siip opgenomen in het addendum-artikel |

## Managementsamenvatting

Bij de registratie in de PEC Zwolle-app wordt de NFC-chip van het paspoort uitgelezen en
worden de ruwe chipbestanden geüpload naar `onboarding.siip.io`: de gezichtsfoto uit de
chip (DG2), de volledige ruwe machineleesbare zone (DG1) en het Document Security Object
(SOD). In die zone zit bij een Nederlands paspoort van vóór 30 augustus 2021 het
burgerservicenummer, dat daarmee aantoonbaar wordt meegestuurd. De server bouwt hieruit een
profiel en stuurt dat terug. De aan de gebruiker getoonde consenttekst noemt maar vijf
velden en dekt deze set niet.

Daarnaast vertrekt telemetrie naar Google al bij de eerste start, vóór enige toestemming en
zonder Consent Mode v2, en laden de web-ticketpagina's volgtechnologie vóór toestemming
zonder cookiebanner. Dezelfde app-architectuur draait als sjabloon bij meerdere clubs, met
een gedeeld foutrapportage-account.

## Aanpak en positie

- Gemeten op een eigen toestel, met een eigen account en een eigen paspoort, uitsluitend op
  eigen netwerkverkeer. Geen toegang tot gegevens van derden, geen ingreep op de backend.
  Nooit een ticket gekocht of geaccepteerd.
- Statische analyse: `apktool` plus `strings`/`grep` op `libapp.so` (Flutter, Dart AOT) en
  `classes.dex` (Kotlin).
- Dynamische meting: een in-process hook op de TLS-schrijf- en leesfuncties (`libssl.so`
  `SSL_write`/`SSL_read`) in een eigen herverpakte app. De app verbindt met de echte server,
  de TLS-handshake en de certificate-pinning slagen normaal, en de payload is plaintext in
  het proces leesbaar. Geen man-in-the-middle, geen gebroken pinning.

De gevoelige hosts (`onboarding`, `iam`, `ticketing`) zijn gepind; het contentkanaal
(`scl.siip.io`) is dat niet. Die differentiële pinning is de reden dat een gewone proxy de
identiteitsupload niet kan lezen en de in-process meting wel.

De afgevangen bestanden staan geredacteerd in [`bewijs/`](bewijs/).

## Bewijsniveaus

**vastgesteld** = aangetoond met gedecodeerde payload of directe capture. **op codeniveau
vastgesteld** = aangetoond uit de binaries. **afgeleid** = gedrag impliceert het.
**open** = niet beslist.

## Ernst

**Kritiek** = bijzondere of wettelijk beschermde persoonsgegevens, of overtreding met brede
impact. **Hoog** = persoonsgegevens of een duidelijke normschending. **Middel** = raakt een
beginsel of grondslag. **Informatief** = context, geen zelfstandige schending.

---

## F-01 · Ruwe paspoortchip (BSN, MRZ, foto, SOD) geüpload bij registratie

| Ernst | Component | Bewijsniveau |
|---|---|---|
| Kritiek | `onboarding.siip.io` | vastgesteld (payload) |

**Bevinding.** Bij registratie stuurt de app een `multipart/form-data`-upload met de ruwe
chipbestanden naar `onboarding.siip.io`.

**Bewijs.** Gedecodeerd uit het afgevangen verkeer (volledig in
[`bewijs/onboarding-upload.txt`](bewijs/onboarding-upload.txt)):

```
POST https://onboarding.siip.io/api/   HTTP/2, TLS 1.3
content-type: multipart/form-data; boundary=5d474873-d117-4399-a7cf-48642da9a3c1

  name="efSodFile"   filename="efSodFile.bin"    application/octet-stream    2.664 byte
  name="photoFile"   filename="photoFile.bin"    application/octet-stream   14.869 byte
  name="dataGroup1"  filename="dataGroup1.bin"   application/octet-stream       93 byte
  name="deviceId"                                                              36 byte
  name="publicKey"                                                            124 byte
```

Dat het model deze datagroepen draagt, blijkt statisch uit `libapp.so`:

```
$ strings -n4 libapp.so | grep -E 'datagroup_[0-9]+_data|MrtdCard'
datagroup_1_data  datagroup_2_data  datagroup_5_data ... datagroup_15_data
MrtdCard.fromJson
package:siip_flutter_bridge/src/models/onboarding/cards/icao_card.dart
```

**Reproductie.** Tijdens de registratie het eigen verkeer meelezen met een in-process hook
op `libssl.so` `SSL_write`/`SSL_read`, gefilterd op `siip.io`, en de multipart-body uit de
HTTP/2 DATA-frames reassembleren.

**Impact.** De sterkste identiteitsgegevens die iemand heeft (staatsdocument plus chip)
verlaten het toestel naar een externe server, al bij registratie.

**Aanknopingspunt.** Dataminimalisatie (art. 5(1)(c) AVG): de volledige ruwe DG1 en het SOD
zijn breder dan functioneel nodig.

---

## F-02 · Het burgerservicenummer wordt meegestuurd

| Ernst | Component | Bewijsniveau |
|---|---|---|
| Kritiek | `onboarding.siip.io` (`dataGroup1`) | vastgesteld (payload) |

**Bevinding.** In de meegestuurde machineleesbare zone (DG1) staat bij een NL-paspoort van
vóór 30 augustus 2021 het BSN in het personal-number-veld.

**Bewijs.** De MRZ (formaat TD3), met alle persoonlijke tekens gemaskeerd:

```
Regel 2, veld voor veld:
  posities  1 t/m 9    documentnummer            ●●●●●●●●●
  posities  11 t/m 13  nationaliteit             NLD
  posities  14 t/m 19  geboortedatum             ●●●●●●
  posities  22 t/m 27  vervaldatum               ●●●●●●
  posities  29 t/m 42  personal-number-veld  ->  BSN (9 cijfers) ●●●●●●●●●   <-- hier
```

Het gemeten personal-number-veld is een negencijferig getal dat de elfproef doorstaat, en
is dus aantoonbaar een BSN, geen willekeurige reeks.

**Impact.** Een uniek overheidsnummer bij een private partij vergroot het risico op
koppeling en identiteitsfraude.

**Aanknopingspunt.** Verwerking van het BSN door een private partij vereist een specifieke
wettelijke grondslag (art. 87 AVG jo. art. 46 UAVG). Het rapport waar Siip zelf naar
verwijst, adviseert het BSN-veld juist technisch niet uit te lezen.

---

## F-03 · Server-side identiteitsprofiel

| Ernst | Component | Bewijsniveau |
|---|---|---|
| Hoog | `onboarding.siip.io` (response) | vastgesteld (payload) |

**Bevinding.** De server bouwt uit de chip-upload een profiel en stuurt dat terug, inclusief
de gezichtsfoto. De belofte dat de gegevens uitsluitend op het toestel blijven, houdt geen
stand.

**Bewijs.** Het echte antwoord (volledig in
[`bewijs/onboarding-response.json`](bewijs/onboarding-response.json)), sleutels onbewerkt,
waarden gemaskeerd:

```json
{
  "organizationUserId": "●●●●●",
  "scopedPersonData": {
    "surname": "●●●●●", "givenNames": "●●●●●", "dateOfBirth": "●●●●●",
    "documentType": "P", "expirationDate": "●●●●●", "issuingStateCode": "NLD",
    "photoType": "jpeg2000",
    "photo":     "<base64-JPEG, 19716 tekens, weggelaten>",
    "photoJpeg": "<base64-JPEG, 27092 tekens, weggelaten>"
  }
}
```

**Impact.** Er ontstaat een centraal identiteitsprofiel met naam, geboortedatum en
gezichtsfoto. Een datalek raakt dan precies die gegevens.

---

## F-04 · Consenttekst dekt de verzonden set niet

| Ernst | Component | Bewijsniveau |
|---|---|---|
| Hoog | app-onboarding | op codeniveau vastgesteld |

**Bevinding.** De app toont vijf gedeelde velden; de feitelijke upload bevat er meer.

**Bewijs.** Uit `libapp.so`:

```
$ strings -n4 libapp.so | grep 'deel je jouw gegevens'
"... Je deelt je voor- en achternaam, roepnaam, foto, geboortedatum en contactgegevens."
```

Niet genoemd, wel verzonden: documentnummer, BSN, nationaliteit, de volledige ruwe MRZ en
het SOD.

**Aanknopingspunt.** Transparantieplicht (art. 13/14 AVG).

---

## F-05 · Profiel ontstaat bij registratie, zonder ticket

| Ernst | Component | Bewijsniveau |
|---|---|---|
| Middel | onboarding vs ticketing | vastgesteld (calltiming) |

**Bevinding.** De identiteitsupload valt vóór enige ticketstap. Er is nooit een ticket
gekocht of geaccepteerd.

**Bewijs.** Calltiming (UTC):

```
10:10:20Z  onboarding.siip.io   upload
10:16:44Z  iam.siip.io          auth
10:19:04Z  mail.siip.io         e-mailverificatie
10:20:15Z  ticketing.siip.io    pas hierna: ticketweergave ophalen (geen aankoop)
```

**Aanknopingspunt.** Noodzaak voor de uitvoering van een ticket-/toegangsovereenkomst
(art. 6(1)(b) AVG) valt weg, want die overeenkomst is er niet.

---

## F-06 · Telemetrie naar Google vóór toestemming, zonder Consent Mode v2

| Ernst | Component | Bewijsniveau |
|---|---|---|
| Hoog | Firebase / GA4 | vastgesteld (emulator) |

**Bevinding.** Bij de eerste start, vóór enig scherm of toestemming, vertrekken al
meetberichten naar Google. Consent Mode v2 ontbreekt.

**Bewijs.** Veldnamen echt, identifier-waarden gemaskeerd:

```
-> firebaseinstallations.googleapis.com   POST .../installations
-> region1.app-measurement.com            POST /a   (GA4)
      events        open_app, screen_view "Introduction App"
      user_property is_registered
      gcs / _dma    afwezig   ->   geen Consent Mode v2
```

**Aanknopingspunt.** Toestemmingseis (art. 6 AVG) en art. 11.7a Telecommunicatiewet;
doorgifte naar de VS (art. 44 e.v. AVG).

---

## F-07 · Foutrapportage naar Sentry vóór toestemming

| Ernst | Component | Bewijsniveau |
|---|---|---|
| Middel | Sentry (VS) | afgeleid (SNI, toestel) |

**Bevinding.** Op het toestel gaat foutrapportage naar Sentry, vóór toestemming. Op
verbindingsniveau (SNI) vastgesteld; de payload zat achter TLS en is niet gelezen.

---

## F-08 · Web-ticketlaag: volgtechnologie vóór toestemming, zonder banner

| Ernst | Component | Bewijsniveau |
|---|---|---|
| Kritiek | ticketwebsites | vastgesteld (browser-scan) |

**Bevinding.** `seizoenkaart.peczwolle.nl` en `wachtlijst.peczwolle.nl` tonen geen
cookiebanner en laden toch volgtechnologie vóór toestemming; een privacyverklaring
ontbreekt. `peczwolle.nl` heeft wel een banner maar vuurt er alsnog overheen.

**Bewijs.** Vóór toestemming actief: Google Analytics 4 (`G-WYHE7T46YP`), een oude Universal
Analytics-property (`UA-43609346-1`), Google Tag Manager (`GTM-TC65FMT`), LinkedIn Ads en
Insight, een Meta-pixel (`id 1769857979961121`, PageView vóór toestemming) en DoubleClick.
Acht volg-cookies gezet vóór enige keuze.

**Reproductie.** Een headless browser-scan in drie standen (zonder interactie, na weigeren,
na accepteren) en de cookies plus uitgaande verzoeken vergelijken.

**Aanknopingspunt.** Art. 11.7a Telecommunicatiewet.

---

## F-09 · Vloot met gedeeld sjabloon en gedeeld foutrapportage-account

| Ernst | Component | Bewijsniveau |
|---|---|---|
| Hoog (systemisch) | `io.siip.saas.<club>` | op codeniveau vastgesteld |

**Bevinding.** De apps van PEC Zwolle, FC Eindhoven, ADO Den Haag en NAC Breda zijn
varianten van hetzelfde white-label sjabloon. FC Eindhoven en PEC delen byte-identieke
onboarding-code, dezelfde `*.siip.io`-hosts, de byte-identieke datagroep-set, en hetzelfde
Sentry-project (organisatie `o4505158983942144`, project `4507100765093888`). Elke club
heeft een eigen Firebase-project. Een ontwerpkeuze werkt daardoor platform-breed door.

De PSV-app is een aparte constructie met dezelfde Siip-module ingebouwd (zie F-11).

---

## F-10 · Jurisdictie: EU-opslag bij een Amerikaanse aanbieder

| Ernst | Component | Bewijsniveau |
|---|---|---|
| Informatief | `*.siip.io` | vastgesteld (DNS) |

**Bevinding.** De functionele hosts resolven naar een AWS Elastic Load Balancer in regio
`eu-west-1` (Ierland).

**Bewijs.**

```
onboarding.siip.io  ->  CNAME  ingress-siip-...eu-west-1.elb.amazonaws.com
                        A       18.203.37.152 , 46.51.174.148   (AWS eu-west-1)
```

**Impact / aanknopingspunt.** De data staat fysiek in de EU, maar bij een Amerikaanse
aanbieder, en valt binnen het bereik van de CLOUD Act.

---

## F-11 · PSV-app: dezelfde identiteitsmodule, plus wallet, advertenties en locatie

| Ernst | Component | Bewijsniveau |
|---|---|---|
| Hoog | `nl.psv` v8.6.0 | op codeniveau vastgesteld (deels gemeten) |

**Bevinding.** PSV gebruikt geen white-label app, maar heeft de Siip-identiteitsmodule in de
eigen clubapp ingebouwd. Endpoints (`onboarding/iam/user/ticketing.siip.io`) en het
datamodel (`MrtdCard`/`IcaoCard`, datagroepen DG1 t/m DG15) zijn identiek aan PEC. Daarmee
is dezelfde identiteits-datastroom bedraad. De app bevat daarnaast een betaal-wallet, een
volledige advertentie-stack en een Cisco-locatie-SDK, in dezelfde app als de
paspoortidentiteit.

**Bewijs.** Een read-only geheugen-scan trof dezelfde profielvelden aan als bij PEC
(`photoJpeg`, `organizationUserId`, `scopedPersonData`, `dateOfBirth`), plus `houseNumber`
en `postalCode`. Gemeten bij app-start (vanaf huis, niet in het stadion):

```
-> api.eu.openroaming.net   POST /sdk/v1/register   (Cisco OpenRoaming / DNA Spaces)
      body: toestel-guid + appId + platform      ->   antwoord: sessietoken
```

De Cisco-SDK provisioneert een wifi-profiel en koppelt de identiteit eraan
(`AssociateUserService`, `installNetworkProfile`). De advertentie-stack omvat AdMob, Criteo,
AppsFlyer, Amazon APS en Meta Audience Network. PSV heeft wel een Didomi-toestemmingsplatform,
waar PEC dat niet had.

**Open.** De payload van PSV loopt over een eigen TLS-laag (UnityTls) en is niet los
afgevangen; de identieke module en endpoints maken de PEC-meting hier van toepassing. Of de
advertentie- en locatie-SDK's vóór of ná de Didomi-toestemming vuren, is niet sluitend
vastgesteld. Geen gezichtsherkennings-SDK aangetroffen.

---

## Openstaande punten

| Punt | Status |
|---|---|
| Bewaartermijn en inzage van het profiel bij Siip | open; vereist controle op schone herinstallatie |
| Transmissie van de advertentie-ID (AD_ID) | open; permissie en plumbing aanwezig, waarde niet gemeten |
| Profilering aan de poort | open; server-side, buiten de app, niet aantoonbaar en niet weerlegd |

## Bijlage A: reproductie

De dynamische meting is reproduceerbaar met de volgende aanpak, op een eigen toestel met een
eigen document:

1. De publieke APK statisch analyseren met `apktool` en `strings`/`grep` op `libapp.so` en
   `classes.dex`, om het datamodel, de endpoints en de consenttekst vast te stellen.
2. Tijdens de registratie het eigen verkeer meelezen met een in-process hook op de
   TLS-schrijf- en leesfuncties (`libssl.so` `SSL_write`/`SSL_read`), gefilterd op `siip.io`.
   De app verbindt met de echte server; de pinning blijft intact en de payload is plaintext
   in het proces leesbaar.
3. De opgevangen bytes per host en richting samenvoegen, de HTTP/2 DATA-frames aflopen en de
   multipart-body en het serverantwoord reassembleren.
4. De web-ticketpagina's los scannen met een browser in drie standen (zonder interactie, na
   weigeren, na accepteren) en de cookies en uitgaande verzoeken vergelijken.

## Bijlage B: bewijs-artefacten en integriteit

De hashes van de gepubliceerde en de verzegelde bestanden staan in
[`SHA256SUMS.txt`](SHA256SUMS.txt). De hashlijst van de oorspronkelijke meting is met
OpenTimestamps in de Bitcoin-blockchain verankerd, zodat de meetdatum onafhankelijk
vaststaat en het bewijs sinds die datum niet ongemerkt te wijzigen is. De ongemaskeerde
originelen blijven verzegeld en niet openbaar.
