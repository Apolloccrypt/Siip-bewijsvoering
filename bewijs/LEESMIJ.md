# De afgevangen bestanden

Dit zijn de daadwerkelijk onderschepte artefacten uit de meting, niet een beschrijving
ervan. Ze komen rechtstreeks uit het eigen netwerkverkeer bij de registratie in de
PEC Zwolle-app. De eigen persoonsgegevens zijn eruit weggelakt; de structuur eromheen is
onbewerkt.

## Wat hier staat

| Bestand | Wat het is |
|---|---|
| [`onboarding-upload.txt`](onboarding-upload.txt) | De echte `multipart/form-data`-upload naar `onboarding.siip.io`, gedecodeerd uit het afgevangen verkeer. De part-headers (veldnamen, bestandsnamen, MIME-types) en de boundary staan er onbewerkt. De inhoud van elk part, de eigen paspoortgegevens, is vervangen door de bytegrootte. |
| [`onboarding-response.json`](onboarding-response.json) | Het echte antwoord dat `onboarding.siip.io` terugstuurde: het server-side profiel. De sleutels zijn onbewerkt, de waarden zijn gemaskeerd, en de twee foto-velden (base64-JPEG) zijn met behoud van hun lengte weggelaten. |
| [`dg2-gezichtsfoto-geblurred.png`](dg2-gezichtsfoto-geblurred.png) | De echte gezichtsfoto uit de paspoortchip (DG2), zoals die in de upload zat, tot een onherkenbare vlek geblurred. |

## Hoe dit te controleren is

De ongemaskeerde originelen zijn verzegeld en niet openbaar. Van elk staat de SHA-256 in
[`../SHA256SUMS.txt`](../SHA256SUMS.txt), en die hashlijst is met OpenTimestamps in de
Bitcoin-blockchain verankerd. Zo staat vast dat de originelen bestonden op de meetdatum en
sindsdien niet zijn gewijzigd. De bestanden hier zijn daaruit afgeleid, met de eigen
persoonsgegevens verwijderd.

De redactie is aan de bron: waar in `onboarding-upload.txt` staat `[WEGGELAKT: 93 byte,
ruwe MRZ (DG1), met het BSN]`, zaten in het origineel op precies die plek 93 bytes ruwe
machineleesbare zone, met daarin het burgerservicenummer. De bytegroottes komen
één-op-één overeen met wat in de artikelen en in [`../rapport.md`](../rapport.md)
staat.
