# PEC Zwolle-app (`io.siip.saas.pec` v2.1.1)

De app is gebouwd met Flutter en geleverd door Siip. Bij registratie leest de app de
NFC-chip van het paspoort uit en uploadt de ruwe chipbestanden naar `onboarding.siip.io`.

## De onboarding-upload, op veldniveau (vastgesteld, payload)

Een `multipart/form-data` POST naar `https://onboarding.siip.io/api/` over HTTP/2.
Gelezen met de in-process TLS-tap (zie [`methodologie.md`](methodologie.md)). De echte,
gedecodeerde upload staat in [`bewijs/onboarding-upload.txt`](bewijs/onboarding-upload.txt),
en het echte serverantwoord in [`bewijs/onboarding-response.json`](bewijs/onboarding-response.json).
De losse, benoemde onderdelen in de upload:

```
POST https://onboarding.siip.io/api/            multipart/form-data, HTTP/2, TLS 1.3

  part  photoFile     application/octet-stream   14.869 byte    DG2, de gezichtsfoto uit de chip (JPEG2000)
  part  dataGroup1    application/octet-stream       93 byte    DG1, de volledige ruwe MRZ, met het BSN
  part  efSodFile     application/octet-stream    2.664 byte    SOD, het Document Security Object (staatshandtekening)
  part  deviceId                                                 toestel-binding
  part  publicKey                                                toestel-sleutel
  part  appBundleId                                              io.siip.saas.pec
```

- **photoFile** is een echt JPEG2000-bestand uit de chip, geen her-gecodeerde pagina-scan.
  Container `ftyp "jp2 "`, afmeting 449 × 599 px, encoderstring `Creator: JasPer Version
  1.900.1`, en een meegereisd XML-metadatablok. Dit is de scherpere chip-biometrie (DG2),
  niet de foto op de pagina van het paspoort.
- **dataGroup1** is de volledige ruwe machineleesbare zone. Bij een Nederlands paspoort
  van vóór 30 augustus 2021 staat het burgerservicenummer in het personal-number-veld
  van die zone. Het is dus meegestuurd.
- **efSodFile** is het Document Security Object: het door de staat ondertekende object
  waarmee de echtheid van het document passief te controleren is.

### Waar het BSN in de MRZ zit (TD3, gemaskeerd)

Een paspoort-MRZ (formaat TD3) is twee regels van 44 tekens. Alle persoonlijke tekens
hieronder zijn vervangen door `●`; alleen de vaste structuur staat er.

```
Regel 1   P<NLD●●●●●●●●●●<<●●●●●●●●<●●●●●<<<<<<<<<<<<<<
Regel 2   ●●●●●●●●●●NLD●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●<<●●

Regel 2, veld voor veld:
  posities  1 t/m 9    documentnummer            ●●●●●●●●●
  positie   10     controlecijfer
  posities  11 t/m 13  nationaliteit             NLD
  posities  14 t/m 19  geboortedatum             ●●●●●●
  positie   20     controlecijfer
  positie   21     geslacht                  ●
  posities  22 t/m 27  vervaldatum               ●●●●●●
  positie   28     controlecijfer
  posities  29 t/m 42  personal-number-veld  →   BSN (9 cijfers) ●●●●●●●●●   ← hier
  positie   43     controlecijfer
  positie   44     totaal-controlecijfer
```

Het gemeten personal-number-veld bevat een negencijferig getal dat de elfproef doorstaat,
de rekenkundige controle waarmee een geldig BSN te herkennen is. Het is dus geen
willekeurig veld van negen cijfers, maar aantoonbaar het BSN.

## Wat de server terugstuurt (vastgesteld, payload)

Het antwoord van `onboarding.siip.io` is een compleet profielobject dat de server
opbouwt uit de geüploade chip-gegevens en terugstuurt naar het toestel. Waarden
gemaskeerd, sleutels echt:

```json
{
  "surname":            "●●●●●",
  "givenNames":         "●●●●● ●●●●●",
  "dateOfBirth":        "●●●●-●●-●●",
  "documentType":       "P",
  "expirationDate":     "●●●●-●●-●●",
  "issuingStateCode":   "NLD",
  "organizationUserId": "●●●●●●●●-●●●●-●●●●-●●●●-●●●●●●●●●●●●",
  "photoType":          "jpeg2000",
  "photoJpeg":          "<base64-JPEG, ~20 kB, weggelaten>",
  "scopedPersonData":   { }
}
```

De foto op het profielscherm komt dus niet uit de lokale scan, maar wordt door de server
samengesteld en teruggestuurd. Dat de identiteit het toestel verlaat en als profiel
terugkomt, is daarmee in beide richtingen gemeten.

## Wat wél en niet in de consenttekst staat

De app toont bij het toewijzen van de kaart letterlijk welke gegevens worden gedeeld.
Uit `libapp.so`:

```
"... Je deelt je voor- en achternaam, roepnaam, foto, geboortedatum en contactgegevens."
"... You share your first and last name, nickname, photo, date of birth and contact details."
```

De feitelijk verzonden set is breder. Per veld:

| Veld / datagroep | Uitgelezen | Verzonden | In consenttekst |
|---|---|---|---|
| Voor- en achternaam | ja | ja (profiel + MRZ) | ja |
| Roepnaam | ja | afgeleid | ja |
| Foto (DG2 chip-biometrie, JPEG2000) | ja | **vastgesteld** (photoFile, 14.869 B) | ja (als "foto") |
| Geboortedatum | ja | **vastgesteld** (in DG1) | ja |
| Contactgegevens (e-mail) | ja | ja (zelf ingevoerd) | ja |
| Documentnummer | ja | **vastgesteld** (in DG1) | **nee** |
| BSN (personal-number-veld DG1) | ja | **vastgesteld** (dataGroup1) | **nee** |
| Nationaliteit | ja | **vastgesteld** (in DG1) | **nee** |
| Volledige ruwe MRZ (DG1) | ja | **vastgesteld** (dataGroup1, 93 B) | **nee** |
| Document Security Object (SOD) | ja | **vastgesteld** (efSodFile, 2.664 B) | **nee** |
| deviceId + publicKey | n.v.t. | **vastgesteld** | **nee** |
| DG5 / DG6 / DG7 / DG11 t/m 15 | model draagt ze | niet in deze upload waargenomen | **nee** |

Het model draagt de datagroepen DG1, DG2, DG5, DG6, DG7 en DG11 tot en met DG15. In de
gemeten upload zaten DG1, DG2 en het SOD. De overige datagroepen zijn in deze meting niet
waargenomen; ze staan als **open**.

## De upload valt vóór enig ticket (vastgesteld)

Calltiming van de meting (UTC), op het eigen toestel:

| Tijd | Host | Betekenis |
|---|---|---|
| 10:10:20Z | onboarding.siip.io | eerste onboarding-call |
| 10:16:43Z | onboarding.siip.io | tweede onboarding-call |
| 10:16:44Z | iam.siip.io | auth |
| 10:19:04Z | mail.siip.io | e-mailverificatie |
| 10:20:15Z | ticketing.siip.io | pas hierna: de app haalt de ticketweergave op |

De onboarding-upload valt ruim vóór de eerste ticketing-call. Er is nooit een ticket
gekocht of geaccepteerd. Het identiteitsprofiel bestaat dus volledig op basis van
registratie alleen, zonder enige toegangsovereenkomst.

## De datastroom in kaart

| Stroom | Ontvanger | Jurisdictie | Moment | Bewijs |
|---|---|---|---|---|
| Identiteit-upload | onboarding.siip.io (AWS eu-west-1) | EU (IE), Amerikaanse aanbieder | bij registratie | **vastgesteld (payload)** |
| Auth | iam.siip.io | EU (IE) | registratie | verkeer vastgesteld, inhoud niet |
| E-mailverificatie | mail.siip.io | EU (IE) | registratie | verkeer vastgesteld, inhoud niet |
| Tickets | ticketing.siip.io | EU (IE) | na registratie | verkeer vastgesteld, inhoud niet |
| Content | scl.siip.io | EU (IE) | doorlopend | vastgesteld (ongepind) |
| Telemetrie | Firebase / Google Analytics | VS | eerste start, vóór toestemming | vastgesteld (emulator) |
| Foutrapportage | Sentry | VS | vóór toestemming | afgeleid (SNI, toestel) |

**Jurisdictie (DNS, reproduceerbaar).** `onboarding.siip.io`, `iam.siip.io` en
`ticketing.siip.io` zijn CNAME-aliassen voor een AWS Elastic Load Balancer in regio
`eu-west-1` (Ierland). De paspoortidentiteit staat dus fysiek in de EU, bij een
Amerikaanse aanbieder, en valt daarmee binnen het bereik van de Amerikaanse CLOUD Act.

## Telemetrie vóór toestemming (vastgesteld, emulator)

Bij de eerste start, zonder dat een scherm is getekend of ergens toestemming voor is
gegeven, vertrekken al meetberichten. Veldnamen echt, identifier-waarden gemaskeerd:

```
Eerste start. Geen interactie. Geen toestemming.

→ firebaseinstallations.googleapis.com   POST /v1/projects/pec-zwolle-fan-app/installations
      appId            <gemaskeerd>
      X-Android-Cert   <app-signing SHA-1, gemaskeerd>
      → antwoord: refreshToken + authToken (persistente installatie-identiteit)

→ region1.app-measurement.com            POST /a   (Google Analytics 4)
      app_instance_id  <install-scoped, gemaskeerd>
      events           open_app, screen_view "Introduction App"
      user_property    is_registered
      google_signals   aanwezig        _npa   1
      gcs / _dma       afwezig   →   geen Consent Mode v2

→ firebase-settings.crashlytics.com       GET  .../apps/io.siip.saas.pec/settings
→ Sentry                                   (op het toestel, vóór toestemming)
```

De enige toestemmingsstap in de app is een akkoord op de voorwaarden bij registratie, in
één blok, zonder aparte keuze per doel. Die stap valt ruim ná deze gegevensstroom.
Consent Mode v2, de standaard van Google om toestemming door te geven, ontbreekt: de
toestemmingssignalen die Google verwacht (`gcs`, `_dma`) zitten niet in de payload.

## De web-ticketlaag (vastgesteld)

Los van de app laden de ticketwebsites al volgtechnologie vóór toestemming.

- **seizoenkaart.peczwolle.nl** en **wachtlijst.peczwolle.nl**: geen cookiebanner, acht
  volg-cookies vóór enige keuze, geen privacyverklaring. Vóór toestemming actief:
  Google Analytics 4 (`G-WYHE7T46YP`), een oude Universal Analytics-property
  (`UA-43609346-1`), Google Tag Manager (`GTM-TC65FMT`), LinkedIn Ads en Insight, een
  Meta/Facebook-pixel (`id 1769857979961121`, PageView vóór toestemming) en DoubleClick.
- **peczwolle.nl**: heeft een Cookiebot-toestemmingsbanner, maar laadt daar overheen
  alsnog Google Tag Manager, reCAPTCHA en DoubleClick vóór toestemming. Zelfde GA4-property
  als de ticketshops.

Dit is een zelfstandige bevinding op de web-laag, met een gewone browser-scan te
reproduceren, los van de app en de chip.

## Wat nog open staat

De grenzen van het bewijs, scherp gehouden. Deze punten zijn gemeten noch weerlegd:

- **Bewaartermijn en inzage.** De upload is gemeten. Hoelang Siip de bestanden bewaart, en
  of ze in te zien zijn of versleuteld liggen met een sleutel die alleen de gebruiker
  heeft, vergt een controle op een schone herinstallatie. Het draaiboek daarvoor ligt klaar.
- **De advertentie-ID (AD_ID).** De app heeft de permissie (`AD_ID`,
  `ACCESS_ADSERVICES_AD_ID`) en de plumbing (`google_signals`, `_npa`) aan boord. Of de
  waarde ook echt wordt verzonden, is op de emulator niet te meten (die heeft geen Play-ad-id)
  en vergt een native capture op het toestel.
- **De poort en profilering daar.** Aan de poort biedt de app een machine-leesbare
  toegangscode aan (`_buildAccessCode`, plus de QR- en streepjescode-generatoren). De
  beslissing om toegang te verlenen of te weigeren gebeurt server-side of in de
  scanner-infrastructuur, buiten de app. Client-side is er geen persoonsgebonden weigering
  (geen stadionverbod- of blacklist-logica) aangetroffen; wat er op de server aan de poort
  gebeurt, is in de app zelf niet aan te tonen en niet te weerleggen.
