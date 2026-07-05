# Siip / PDT: bewijsvoering

Reproduceerbare en verifieerbare bewijsvoering bij het privacyonderzoek naar
Persoonlijke Digitale Toegang (PDT) van Siip.

Deze repository is de technische onderbouwing bij de volgende publicaties op mickbeer.com:

- **Het onderzoek van 1 juli 2026**: [PEC Zwolle, FC Eindhoven, ADO en NAC willen je paspoort](https://mickbeer.com/artikelen/siip-pdt-pec-zwolle-paspoort-toegang/)
- **Het addendum met wederhoor van 3 juli 2026**: [Bij registratie vertrekken je BSN en je pasfoto uit de chip naar de server van Siip](https://mickbeer.com/artikelen/siip-addendum-wederhoor/)
- Toekomstige artikelen over hetzelfde onderwerp worden hier aangevuld.

Het doel is transparantie. Iedereen kan de technische bevindingen zelf naslaan,
de gemeten datastromen stap voor stap volgen, en de vaste bewijs-artefacten tegen
hun SHA-256 controleren. De conclusies in de artikelen rusten op wat hier staat.

## De bevinding staat vast; de waarden staan hier niet

Voor alle duidelijkheid: het onderzoek toont juist **aan** dat het burgerservicenummer,
de volledige machineleesbare zone en de gezichtsfoto uit de chip worden verstuurd naar
de server van Siip. Dat is de kern van de bevinding, niet iets wat ontbreekt.

Wat deze repository bewust **niet** doet, is die persoonlijke waarden opnieuw publiceren.
Het gaat om een eigen paspoort, en het echte BSN, de ruwe zonegegevens en de eigen
gezichtsfoto horen niet op straat. Wat hier staat is de techniek eromheen: veldnamen,
bestandsgroottes, MIME-types, endpoints, hashes en de structuur van elk veld, met de
persoonlijke tekens vervangen door `●`. Zo is bijvoorbeeld te zien dát het BSN in het
personal-number-veld van de MRZ zit en wordt meegestuurd, zonder dat het nummer zelf
leesbaar is.

De ruwe onderschepte bestanden, waarin de gegevens wél leesbaar staan, zijn verzegeld en
niet openbaar. Van elk staat alleen de SHA-256 in [`SHA256SUMS.txt`](SHA256SUMS.txt).
Wie een verzegelde kopie heeft, kan die tegen die hash controleren en zo vaststellen dat
het om exact hetzelfde bewijs gaat. De hele bevinding is daarmee te verifiëren zonder dat
er persoonsgegevens openbaar worden.

## Onderzoekspositie

- Gemeten op een eigen toestel (OUKITEL WP19, Android 12) en een Android-emulator,
  met een eigen account en een eigen paspoort, uitsluitend op eigen netwerkverkeer.
- Statische analyse op de publiek verkrijgbare app-binaries.
- Geen toegang tot gegevens van derden. Geen ingreep op de systemen of de backend van
  Siip. Nooit een ticket gekocht of geaccepteerd.
- Het meten en vastleggen van het eigen netwerkverkeer op een eigen toestel is een
  meting van wat de app met de eigen gegevens doet. Het is geen beveiligingsonderzoek
  op andermans systeem en er is geen kwetsbaarheid misbruikt.

De reactie van Siip op de bevindingen is integraal opgenomen in het
[addendum-artikel](https://mickbeer.com/artikelen/siip-addendum-wederhoor/). De hier
opgenomen bevindingen zijn feitelijke waarnemingen uit meting en code; de
juridische duiding staat als aanknopingspunt benoemd, niet als vaststaand oordeel.

## Bewijsniveaus

Elk punt in de documenten draagt een niveau, zodat waarneming en gevolgtrekking uit
elkaar blijven:

- **vastgesteld**: aangetoond met gedecodeerde payload of directe capture.
- **op codeniveau vastgesteld**: aangetoond uit de app-binaries (ontwerp of gegevensmodel).
- **afgeleid**: het gedrag of de architectuur impliceert het, de payload zelf is niet gezien.
- **open**: niet beslist; de vervolgmeting staat benoemd.

## De afgevangen bestanden

De daadwerkelijk onderschepte data, geen beschrijving ervan, staat in
**[`bewijs/`](bewijs/)**:

- [`bewijs/onboarding-upload.txt`](bewijs/onboarding-upload.txt) — de echte
  `multipart/form-data`-upload naar `onboarding.siip.io`, met de part-headers en de
  boundary onbewerkt en de eigen paspoortgegevens per part weggelakt.
- [`bewijs/onboarding-response.json`](bewijs/onboarding-response.json) — het echte
  server-side profiel dat terugkwam, sleutels onbewerkt, waarden gemaskeerd.
- [`bewijs/dg2-gezichtsfoto-geblurred.png`](bewijs/dg2-gezichtsfoto-geblurred.png) — de
  echte gezichtsfoto uit de chip, tot een onherkenbare vlek geblurred.

Zie [`bewijs/LEESMIJ.md`](bewijs/LEESMIJ.md) voor hoe deze tegen de verzegelde originelen
te controleren zijn.

## Toelichting per app

| Bestand | Onderwerp |
|---|---|
| [`methodologie.md`](methodologie.md) | Hoe er is gemeten: statische analyse en de in-process TLS-tap zonder man-in-the-middle. |
| [`pec-zwolle.md`](pec-zwolle.md) | PEC Zwolle-app: de onboarding-upload op veldniveau, de MRZ-structuur, het server-profiel, de web-ticketlaag. |
| [`psv.md`](psv.md) | PSV-app: dezelfde Siip-identiteitspijplijn, plus wallet, advertentie-stack en de Cisco-locatie-SDK. |
| [`siip-platform.md`](siip-platform.md) | Het platform en de vloot: het white-label sjabloon, de gedeelde infrastructuur en de differentiële pinning. |
| [`terminologie.md`](terminologie.md) | De vaktermen en afkortingen in gewone taal. |
| [`SHA256SUMS.txt`](SHA256SUMS.txt) | De SHA-256 van de bestanden en de verzegelde originelen. |

## Verifiëren

De vaste artefacten (de app-binaries en de meetscripts) zijn met hun SHA-256 te
controleren:

```
sha256sum -c SHA256SUMS.txt
```

De verzegelde captures zijn niet in deze repository opgenomen omdat ze ongemaskeerde
persoonsgegevens bevatten. Hun SHA-256 staat wel vermeld. De hashlijst van de
oorspronkelijke meting is met OpenTimestamps in de Bitcoin-blockchain verankerd, zodat
onafhankelijk te controleren is dat het bewijs sinds de meetdatum niet is gewijzigd.
