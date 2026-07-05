# PSV-app (`nl.psv` v8.6.0)

PSV gebruikt geen white-label Siip-app. PSV heeft de Siip-identiteitsmodule in de eigen
clubapp gebouwd. De app is een Unity-app; de Siip-flow zit in een Unity/C#-laag.

Het merendeel hieronder is **op codeniveau vastgesteld** (aanwezig in de binaries).
Een deel is on-device gemeten. Wat statisch is, is geen bewijs dat het ook vertrekt of
vóór toestemming vuurt; dat staat er per punt bij.

## Dezelfde Siip-identiteitspijplijn (op codeniveau vastgesteld)

Namespace `io.siip.packages.plugin.androidunityinterface.services`:

- identiteit: `OnboardingService`, `ExternalOnboardingService`, `PersonService`,
  `AuthService`, `MailService`, `LoginWithSiipService`
- ticketing: `TicketService`, `TicketBundleService`, `OfferService`, `WalletService`
- overig: `InitializationService`, `CiscoService`

De endpoints zijn identiek aan PEC: `onboarding.siip.io`, `iam.siip.io`, `user.siip.io`,
`ticketing.siip.io`. Het datamodel is identiek: `MrtdCard`/`IcaoCard` met de datagroepen
DG1, DG2, DG5, DG6, DG7 en DG11 tot en met DG15, byte-voor-byte dezelfde set als PEC.
Permissies: `NFC` + `CAMERA` (paspoortscan) + `AD_ID`.

Daarmee is de datastroom die bij PEC op netwerkniveau is bewezen (BSN + DG2 + SOD naar
`onboarding.siip.io`, met een server-side profiel terug) in de PSV-app bedraad met
dezelfde code en dezelfde endpoints. De live-payload zelf loopt bij PSV over een eigen
TLS-laag (UnityTls, buiten `libssl.so`) en is niet met de PEC-methode meegelezen; de
identieke module en endpoints maken de PEC-meting hier van toepassing.

Een read-only geheugen-scan tijdens gebruik trof de veldnamen `photoJpeg`,
`organizationUserId`, `scopedPersonData` en `dateOfBirth` aan, identiek aan het
PEC-profielobject, plus `houseNumber` en `postalCode` (adres). Dit bevestigt het
datamodel; het zijn de veldnamen, niet de live waarden.

## Waar PSV verder gaat dan PEC

### Siip als identiteitsbroker (op codeniveau vastgesteld)

`LoginWithSiipService` met `loginWithSiipAuthorize/Deny`, `startScan/stopScan` en
`LoginWithSiipAuthorizeRequest`. Een QR-gebaseerde cross-device login-autorisatie, in de
vorm van "log in met Siip". De identiteit uit het paspoort wordt zo een inlogmiddel voor
andere diensten.

### Betaal-wallet in dezelfde app als de paspoortidentiteit (op codeniveau vastgesteld)

`WalletService`, met Stripe, Adyen, iDEAL en TopUp. Identiteit en betaling zitten in één
app.

### Advertentie-stack (op codeniveau vastgesteld)

In dezelfde app als de paspoortidentiteit en de wallet:

- Google AdMob
- Criteo (retargeting)
- AppsFlyer (attributie)
- Amazon APS / A9 (header bidding)
- Meta Audience Network
- DoubleClick / googleadservices

### Cisco-locatie-SDK: identiteit-gekoppelde in-venue presence (gemeten + code)

`com.cisco.or.sdk` (Cisco OpenRoaming / DNA Spaces). De SDK provisioneert een
WiFi-Passpoint/Hotspot-profiel op het toestel (automatisch verbinden met stadionwifi) en
koppelt de identiteit aan dat profiel (`AssociateUserService`, `installNetworkProfile`,
`DeleteProfileService`).

**Gemeten, vanaf huis (niet in het stadion):** bij app-start doet de app een registratie
bij `api.eu.openroaming.net` (`POST /sdk/v1/register`) met een toestel-guid, en Cisco
antwoordt met een sessietoken. Het toestel schrijft zich dus bij elke app-start in bij een
Amerikaans locatie-analyseplatform. De SDK kent regionale endpoints (EU, globaal, APAC);
de gemeten registratie ging naar de **EU-regio**. Er is geen verkeer naar de APAC-regio
waargenomen.

De inschrijving is gemeten. Actieve locatietracking en de koppeling van de guid aan een
naam zijn **afgeleid** uit de code, niet uit het verkeer.

### Consent (rijper dan PEC, maar open)

PSV heeft een echte Didomi-CMP (IAB TCF v3), waar PEC er geen had. In de meting vuurden de
trackers ná de consent-klik, dus de CMP lijkt hier te gaten. Of AdMob, Criteo, AppsFlyer,
Amazon APS, Meta en Cisco allemaal ná de Didomi-toestemming vuren, is per SDK niet
sluitend vastgesteld en staat als **open**.

## Consistent met PEC (geen overclaim)

Geen gezichtsherkennings-SDK aangetroffen. Alleen de bekende paspoort/NFC-componenten en
de advertentie- en analyse-SDK's. De biometrie-lijn blijft daarmee ook hier: gezichts­
herkenning is niet aantoonbaar.

## Open

- Welke datagroepen daadwerkelijk vertrekken bij PSV (het model draagt DG1–15; de
  bewezen PEC-upload bevatte DG1, DG2 en het SOD). Vereist een on-device payload-capture
  op de UnityTls-laag.
- De consent-timing van de advertentie- en locatie-SDK's ten opzichte van Didomi.
- De koppeling van de Cisco-guid aan de identiteit.

## Gedeelde infrastructuur

De PSV-app rapporteert fouten naar hetzelfde gedeelde Siip-Sentry-project als PEC en
FC Eindhoven. Zie [`siip-platform.md`](siip-platform.md).
