# Het Siip-platform en de vloot

De bevinding bij één club staat niet op zichzelf. Siip levert een white-label sjabloon en
rolt dat per club uit.

## Eén sjabloon, meerdere clubs (op codeniveau vastgesteld)

De pakketnaam volgt het patroon `io.siip.saas.<club>`. Bevestigd in Google Play:
`io.siip.saas.pec` (PEC Zwolle), `io.siip.saas.fceindhoven` (FC Eindhoven),
`io.siip.saas.adodenhaag` (ADO Den Haag) en `io.siip.saas.nac` (NAC Breda).

FC Eindhoven en PEC Zwolle zijn naast elkaar gelegd:

- identieke `fan_shell` en `siip_flutter_bridge`
- byte-identieke datagroep-set {DG1, DG2, DG5, DG6, DG7, DG11 t/m 15}
- identieke `*.siip.io`-hostenset (onboarding, iam, user, ticketing, mail, plus staging en scl)
- dezelfde tracking-permissies, geen toestemmingsmodule (CMP)

De code die het paspoort uitleest en verstuurt is dus dezelfde. De onboarding-architectuur
die bij PEC op netwerkniveau is gemeten, is het sjabloon dat de vloot draait. PSV is een
aparte constructie: geen white-label, maar dezelfde Siip-module ingebouwd in de eigen
clubapp (zie [`psv.md`](psv.md)).

## Gedeeld foutrapportage-kanaal (op codeniveau vastgesteld)

PEC, FC Eindhoven en PSV rapporteren fouten naar hetzelfde Sentry-project: organisatie
`o4505158983942144`, project `4507100765093888`. De crashmeldingen van meerdere clubs
lopen dus door één gemeenschappelijk Siip-account. Elke club heeft daarnaast een eigen
Firebase-project, dus de analytics zijn per club gescheiden.

## Differentiële certificate pinning (vastgesteld)

Niet alles is even zwaar beveiligd, en dat verschil is verhelderend.

- **Gepind** (weigert elke tussenpartij): `onboarding.siip.io` (identiteit),
  `iam.siip.io` (auth), `user.siip.io` (profiel), `ticketing.siip.io` (tickets).
- **Niet gepind** (leesbaar met een gewone proxy): `scl.siip.io`, het contentkanaal met
  nieuws, fixtures en afbeeldingen.

De gevoelige kanalen zijn dus bewust extra beveiligd. Dat is een sterke keuze. Het is ook
de reden dat de identiteits-upload alleen met een in-process tap te lezen was, en niet met
een gewone proxy (zie [`methodologie.md`](methodologie.md)).

## Jurisdictie (vastgesteld, reproduceerbaar via DNS)

De functionele `*.siip.io`-hosts wijzen naar een AWS Elastic Load Balancer in regio
`eu-west-1` (Ierland). De gegevens staan fysiek in de EU, bij een Amerikaanse aanbieder,
en vallen daarmee binnen het bereik van de Amerikaanse CLOUD Act. De telemetrie gaat
daarnaast naar Google en Sentry (VS-entiteiten).

## Ambitie

De publieke ambitie van de KNVB is dat PDT vanaf seizoen 2027/28 de standaard wordt bij
alle Nederlandse clubs. Een ontwerpkeuze in het sjabloon werkt dan platform-breed door.

## Feitelijke aanknopingspunten (geen vaststaand oordeel)

- **Transparantie (art. 13/14 AVG):** de feitelijk verzonden set is breder dan de aan de
  gebruiker getoonde consenttekst.
- **Dataminimalisatie (art. 5(1)(c)):** de volledige ruwe MRZ en het SOD worden geüpload,
  niet enkel de functioneel benodigde velden.
- **Grondslag (art. 6):** het profiel ontstaat bij registratie zonder ticket.
- **BSN:** verwerking van het BSN door een private partij vereist een specifieke
  wettelijke grondslag; hier reist het mee in de ruwe MRZ.
- **Telecommunicatiewet art. 11.7a:** volgtechnologie vóór toestemming op de web-ticketlaag.

De reactie van Siip op deze bevindingen staat integraal in het
[addendum-artikel](https://mickbeer.com/artikelen/siip-addendum-wederhoor/).
