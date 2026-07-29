# Bus Whisky - Overdrachtsdossier / Runbook

Doel van dit document: de ICT-partij (borger) in een keer laten zien welke geautomatiseerde systemen er draaien, hoe ze samenhangen, waar ze staan, en waar je moet zijn om mee te kijken of in te grijpen. Bevat geen wachtwoorden of API-sleutels; alleen namen, locaties en ID's.

Laatst bijgewerkt: 29 juli 2026.

---

## 1. In het kort

De hele AI- en automatiseringslaag van Bus Whisky draait in twee systemen:

1. **GitHub** houdt de "canon" bij: een tekstbestand dat de enige goedgekeurde bron van waarheid is over Bus Whisky (prijzen, regels, teksten, links). Alle AI-agents lezen hieruit.
2. **n8n Cloud** draait alle automatisering: de website-chatbot, de mailverwerking, koppelingen met het reserveringssysteem (KMT) en het boekingssysteem (Formitable), en tientallen onderhouds- en synchronisatietaken.

Daaromheen hangen externe diensten (Microsoft 365, Anthropic, Formitable, KMT, Nappkin, WordPress, Google, Meta, Brevo) die n8n via credentials aanspreekt.

---

## 2. Systemen en waar ze staan

| Systeem | Waar | Waarvoor |
|---|---|---|
| n8n Cloud | https://buswhiskyevents.app.n8n.cloud | Alle automatisering (workflows). Project: **Bus Whisky Events <info@buswhisky.com>** (id `l1xkl4LkW85379Rb`), 94 workflows. |
| GitHub - canon | repo `dhurkmans716/buswhisky-canon`, bestand `canon.md` (publiek) | Enige bron van waarheid voor alle AI-agents. |
| GitHub - data | repo `dhurkmans716/buswhisky-data` | Beschikbaarheidsfeed (JSON + XML) voor chatbot en frontend, elk uur ververst. |
| Google-account | `d.hurkmans716@gmail.com` | Eigenaar van de GitHub-repos en van de Google-diensten (Business Profile, Cloud, Ads, Search Console). |
| Microsoft 365 / Outlook | mailboxen `info@buswhisky.com`, `info@heischehoeve.nl`, `dennis@`, `canon@`, `anja@`, `housekeeping@` | Bron en bestemming van alle mailautomatisering (via Microsoft Graph). |
| Formitable | `api.formitable.com/api/v1.2`, restaurant-id `59f03a82` | Restaurant-, B&B-, rondleiding- en proeverijboekingen + beschikbaarheid. |
| KMT | intern reserveringssysteem (login via n8n) | Offertes, orders, bedrijfsuitjes, vergaderingen, bruiloften, planning. |
| Anthropic API | via n8n-credential | Het taalmodel (Claude) achter chatbot, mailagents en vertaling. |
| Nappkin | kassasysteem restaurant | Daglijsten en omzetdata. |
| WordPress | buswhisky.com / heischehoeve.com | Website; wordt gelezen/geschreven voor content en vertalingen. |
| Meta (Facebook/Instagram) | Graph API | Social-reactie-agent. |
| Brevo | e-mailmarketing | Nieuwsbrieven/campagnes. |

---

## 3. De canon (kern van alles)

- De canon is `canon.md` in `dhurkmans716/buswhisky-canon`. Het is de enige goedgekeurde bron; wijkt iets elders af, dan is de canon leidend.
- Agents lezen de canon live op twee manieren: via de GitHub raw-URL (met circa 5 minuten cache) of via de GitHub contents-API (verser).
- **Versiebeheer zit gratis in GitHub:** elke wijziging is een commit met volledige historie. Terugdraaien = de betreffende commit terugzetten. Dit is meteen je belangrijkste vangnet voor de inhoud.

### Hoe de canon geschreven wordt

De canon wordt nooit door een AI overgetypt (dat zou corruptie geven), maar altijd deterministisch bewerkt: een proces haalt het bestand op, past exacte tekstvervangingen toe in een Code-node en commit direct. Schrijvers naar de canon zijn onder andere:

- `Bus Canon Mailcorrectie` (id `tqtS20vuQY7z91yM`): correcties per mail op `canon@buswhisky.com`.
- `Formitable Deeplink Sync` (id `RDbYgP2Dk1J4swgL`): dagelijkse boekingslinks in sectie 11.
- `Theatershow Canon Sync` (id `y3Y7BIPu5zSodK7o`) en `Camping Culinair datum sync` (id `eoTAcnRRpyjF4QVA`): actuele data in hun markerblokken.
- `Canon wekelijkse aanvulling` (id `LRCtctaKawfEiik0`): stuurt alleen voorstellen per mail, schrijft zelf niet.

Alle schrijfacties gebruiken de credential **Bus Canon Schrijver** (githubApi).

---

## 4. n8n: de belangrijkste workflows

Er staan 94 workflows in het project. De volledige lijst zie je in n8n zelf; hieronder de kritische kern, gegroepeerd per functie.

**Klantcontact (live naar buiten):**
- `Bus Whisky - Website Chatbot` (id `HpqZf18IbJHU4StM`, ACTIEF): de publieke website-chatbot. Leest de canon live, beantwoordt met Claude Sonnet, boekt zelf restauranttafels en stuurt boekingsdeeplinks. Logt gesprekken in Data Table `Chatgesprekken log` (id `5v3IHlQcXooglRk3`).
- `Bus Whisky - WhatsApp Chatbot` (id `8VQpW5tDkldlHcq1`, INACTIEF tot Meta/WhatsApp-credentials rond zijn).
- `Bus Whisky - Beschikbaarheid API (Bonnie/extern)` (id `LxmId8vpo0dZGMzg`, ACTIEF): webhook waarmee de AI-telefoniste Bonnie live beschikbaarheid opvraagt.

**Chatbot-subworkflows (boeken/checken):**
- Restaurant boeken: `EI2m01yV04FQzfbF`. Restaurant beschikbaarheid: `MLff4sz3JRmR1g79`. Uitjes beschikbaarheid: `kZeUalk9UKyCvvH2`. B&B beschikbaarheid: `ARLekw1dfuE4BRJl`.

**Mailverwerking:**
- `Mail-brein v1 (classificatie + router)` (id `AIzboLkS6h34JJhq`, ACTIEF): centraal brein voor `info@heischehoeve.nl` en `info@buswhisky.com`. Classificeert inkomende mail en routeert naar de juiste actie.
- `BUS Mailbox Agent - concept-antwoorden` (id `drNT4K0jWD2WYucl`): schrijft concept-antwoorden in Outlook, verstuurt nooit automatisch.
- Diverse opschoners, afmelders en doorstuur-workflows, gebundeld onder `Opschoner - info@HH + info@BUS` (id `xWgFjLAwNw7QzgaO`, elke 30 min).

**Koppelingen en syncs:**
- `BUS Beschikbaarheid Publisher (KMT naar GitHub)` (id `PHrrKEYt1XNCTztJ`, ACTIEF): publiceert elk uur beschikbaarheid naar de data-repo.
- `KMT <-> Formitable dagelijks` (id `6BDofv434twxSUf5`, ACTIEF): houdt boekingen tussen KMT en Formitable gelijk.
- `KMT Arrangementen sync` (id `kFagO9bZlCGQPeBb`), `KMT offerte-automatisering` (id `KYnF8FGTlcMrJupS`), en de canon-syncs uit sectie 3.

**Overig:** website-vertaling NL naar DE/EN (`BW Vertaling - PRODUCTIE`, id `XRKCR29H8KuVPdyD`), sales-heractivatie, housekeeping- en daglijstrapporten, social-reactie-agent (id `xpiW1D2r3yws6REV`).

Let op: workflows met TEST, PROBE, TEMP of "mag weg" in de naam zijn hulp- of testworkflows en niet operationeel kritisch.

---

## 5. Credentials (namen en typen, geen geheimen)

In n8n staan onder dit project de volgende credentials. De geheime waarden zijn nooit zichtbaar via de API; ze staan versleuteld in n8n.

| Naam | Type | Waarvoor |
|---|---|---|
| Microsoft Outlook account | microsoftOutlookOAuth2Api | Mailboxen lezen/schrijven (hoofdkoppeling, id `04oSLlCD8vYSLJfZ`). |
| Microsoft Outlook account 2 / OAuth2 API (x2) | microsoftOutlookOAuth2Api | Extra mailbox-koppelingen. |
| Anthropic account (x2) | anthropicApi | Claude-model voor de agents. |
| Bus Canon Schrijver | githubApi | Schrijfrechten op de canon-repo (id `IYIxDPo2f6hyLKLv`). |
| GitHub OAuth2 API (x2) | githubOAuth2Api | Overige GitHub-acties. |
| Header Auth account | httpHeaderAuth | Formitable API-sleutel (id `ykHPYF20uhlyp3iL`). |
| KMT login | httpCustomAuth | Inloggen op het KMT-reserveringssysteem. |
| WordPress buswhisky | httpBasicAuth | Website lezen/schrijven. |
| Driver login | httpBasicAuth | driver.buswhisky.com. |
| Facebook Graph account (x2) | facebookGraphApi | Social-reactie-agent. |
| Custom Auth account | httpCustomAuth | Overige API-authenticatie. |

Kortom: alle geheimen zitten in n8n (credentials) en in de betreffende externe diensten zelf. Er staan geen sleutels los in workflows, op een enkele legacy-node na die in de workflowomschrijving wordt genoemd.

---

## 6. Belangrijkste ID's (naslag)

- n8n-project: `l1xkl4LkW85379Rb`
- Storingsalarm-workflow: `LjSODUqe2O3a8MLI` (zie sectie 7)
- Chatbot: `HpqZf18IbJHU4StM` - Mail-brein: `AIzboLkS6h34JJhq`
- Canon-repo: `dhurkmans716/buswhisky-canon` (`canon.md`) - Data-repo: `dhurkmans716/buswhisky-data`
- Formitable: `api.formitable.com/api/v1.2`, restaurant `59f03a82`
- Data Table chatlog: `5v3IHlQcXooglRk3`
- Canon-schrijfcredential: `Bus Canon Schrijver` (`IYIxDPo2f6hyLKLv`)

---

## 7. Monitoren, ingrijpen en herstellen

**Meekijken wat er gebeurt:**
- In n8n: open een workflow en bekijk het tabblad Executions. Daar staan alle runs met status (geslaagd/mislukt) en de volledige data per node. Dit is de plek om een storing te onderzoeken.

**Ingrijpen bij een probleem:**
- Een workflow pauzeren: zet de toggle Active uit (of unpublish). De workflow stopt dan direct met draaien; de rest blijft werken.
- Een verkeerde canon-wijziging terugdraaien: zet in GitHub de betreffende commit terug (revert). De agents lezen daarna binnen enkele minuten weer de goede versie.

**Storingsalarm (nieuw):**
- Workflow `BUS Storingsalarm (n8n foutmelder)` (id `LjSODUqe2O3a8MLI`) mailt automatisch zodra een gekoppelde workflow een run laat mislukken. De mail bevat de naam van de workflow, de foutmelding, de laatste node en een directe link naar de mislukte executie.
- De mail gaat nu naar `dennis@buswhisky.com`. **Voeg hier het adres van de ICT-partij aan toe** (in de node "Mail storing naar beheer", veld toRecipients).
- Gekoppeld aan het alarm (foutafhandelaar ingesteld): de chatbot, het mail-brein, de beschikbaarheid-publisher, de Bonnie-API, de KMT<->Formitable-sync en de drie canon-schrijvers. Voor elke andere workflow kun je hetzelfde instellen via Workflow-instellingen -> Error Workflow -> "BUS Storingsalarm".

**Back-up:**
- De canon is via GitHub al volledig geborgd (versiehistorie + terugdraaien).
- n8n Cloud houdt van elke workflow een eigen versiegeschiedenis bij (per workflow terug te zetten).
- Voor een kopie buiten n8n (aanbevolen als extra vangnet): een wekelijkse export van alle workflows als JSON naar GitHub of Drive. Dit vraagt eenmalig een n8n API-sleutel (aan te maken in n8n onder Settings -> API). Zodra die er is, kan dit als terugkerende workflow worden ingericht.

---

## 8. Toegang regelen voor de ICT-partij (borger)

Om mee te kunnen kijken en in te grijpen heeft de ICT-partij het volgende nodig:

1. **n8n:** nodig hen uit als lid van de n8n-omgeving/het project, met een rol waarmee ze workflows kunnen zien, bewerken, de executies bekijken en workflows pauzeren/deactiveren.
2. **GitHub:** voeg hun account toe als collaborator op `buswhisky-canon` (en `buswhisky-data`). Dan zien ze de canon, de wijzigingshistorie, en kunnen ze terugdraaien.
3. **Externe accounts** (Microsoft 365, Anthropic, Formitable, KMT, Google, Nappkin, WordPress): deel deze via een wachtwoordmanager met afgeschermde toegang, nooit losse persoonlijke wachtwoorden.

---

## 9. Rolafspraak (de borger-rol)

Voorstel voor een heldere verdeling:

- **Dennis (eigenaar):** bepaalt de inhoud en het beleid (wat er in de canon staat, welke automatisering er komt).
- **ICT-partij (borger):** bewaakt de techniek, kijkt mee via de executies en het storingsalarm, en mag zelfstandig ingrijpen bij een storing (een workflow pauzeren, een canon-wijziging terugdraaien). Inhoudelijke wijzigingen aan canon of workflows gaan in overleg.

Leg vast wat de ICT-partij zelfstandig mag (ingrijpen bij storing) en wat in overleg gaat (structurele wijzigingen), zodat er bij een incident geen twijfel is over wie wat doet.
