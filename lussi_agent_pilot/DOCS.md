# Lussi agent

Spojuje dům s Lussi cloudem (`api.lussivoice.com`):

- **identita centrály** – klíč Ed25519 vzniká při prvním startu a nikdy neopustí `/data` doplňku;
- **párovací kód** `LUS-XXXXX-XXXXX` v panelu doplňku (jen pro přihlášené v HA);
- **relay** – jedno odchozí šifrované spojení do cloudu pro telefony na dálku;
- **brána doma** na `https://homeassistant.local:8444`: s certifikátem z `/ssl/lussi_doma`, když ho dům má
  (stejný jako NGINX na 8443), jinak si agent certifikát vyrobí sám v `/data/brana` pro adresy centrály v síti
  domu a při změně adresy ho obnoví se stejným otiskem. Panel ukazuje, odkud certifikát je, jeho adresy a otisk.
  Telefon v jiné síti než centrála (VLAN, NAT) se domů připojí přes relay.

Nabu Casa, NGINX ani přesměrování portů nepotřebuje; když je dům má, zůstávají beze změny.

## Instalace

Nastavení → Doplňky → Obchod doplňků → ⋮ → Repozitáře → přidat `https://github.com/pmisun/lussi-voice-doplnky`
→ Lussi agent → Instalovat → Spustit. Pak otevřít panel **Lussi agent** a párovací kód zadat v Aplikaci
Lussi Voice. Volby níže mají výchozí hodnoty, které stačí na začátek; zkontrolovat je správce domu.

K Home Assistantu se agent připojuje přímo za každého člena jeho tokenem na `http(s)://homeassistant:<port>`.
Port a SSL zjistí při startu ze Supervisoru (`/core/info`, proto `hassio_api` s výchozí rolí); když
Supervisor neodpoví, zkusí 8123. Adresu ukazuje panel doplňku a log při startu.

Nového člena ze seznamu (správce, dospělý, dítě i host) agent **propojí sám**: založí mu uživatele HA bez práv
správce (bez hesla, jen z místní sítě), propojí ho s osobou a vydá token do `/data`. Odebranému členovi
uživatele vypne (nemaže). K tomu používá proxy Supervisoru (`homeassistant_api`) s pevným seznamem
příkazů správy uživatelů a osob; podrobnosti a dopad v
`docs/adr/0001` v repozitáři agenta.
Dítě a host vidí a ovládají jen své místnosti (`mistnosti` ze seznamu členů; host bez místností celý
dům mimo bezpečí, skripty a topení). Host spouští jen scény z nabídky domu (ne ty, které sahají na bezpečí),
skripty nevidí a topení, ohřev vody a vlhkost ovládá jen ve svých místnostech. Dítě nikdy neovládá vrata, zámky ani alarm a píše vždy dětskému agentovi. Hostovi
na konci přístupu brána zavře spojení a agent vypne jeho uživatele HA. Stav propojení vidí správce
v aplikaci a v panelu doplňku.

## Nastavení

| Volba | Význam |
| --- | --- |
| `cloud_url` | adresa Lussi cloudu, výchozí `https://api.lussivoice.com` |
| `bezpeci_navic` | spínače a skripty, které otevírají vrata, bránu nebo dveře, odemykají nebo vypínají alarm (entity mimo domény bezpečí); brána je odmítne všem, a když nejsou zámek, alarm ani kryt, i skryje |
| `bezpeci_navic_zkontrolovano` | správce potvrdil, že seznam `bezpeci_navic` je úplný (i prázdný); výchozí ne. Dokud není zaškrtnuté, brána spínače, skripty a tlačítka nepustí nikomu (kromě skriptů domu Lussi s pevnými poli) a panel i log na to upozorní |
| `detsky_agent` | agent konverzace dětského pokoje (`conversation.…`); psaní Lussi dítěte jde vždy jemu. Prázdné (výchozí) = dům dětského agenta nemá a dítě Lussi z aplikace nepíše |
| `televize` | televize Naboo (`media_player.…`); dítě a host s místnostmi ji z aplikace ovládají, jen když je v jejich místnostech. Prázdné (výchozí) = dům televizi Naboo nemá; dítě a host s místnostmi ji pak neovládají |
| `nakupni_seznam` | nákupní seznam domácnosti (`todo.…`, výchozí `todo.shopping_list`); dítě vidí jen tento seznam, i mimo své místnosti, a smí do něj přidávat a odškrtávat; host a servis žádný seznam |
| `servis_tunel_cil` | servisní tunel technika Lussi (K147): `host:port` doplňku SSH ve vnitřní síti doplňků, např. `a0d7b954-ssh:22`; prázdné = tunel vypnutý; port 22222 nikdy. Port doplňku SSH do sítě nevystavovat, jediná cesta je tunel v okně servisu |
| `servis_doplnky` | slugy doplňků, které smí technik ve schváleném servisu restartovat a číst jejich log (např. `lussi_agent`) |
| `servis_komponenty` | entity `update.…` komponent Lussi, které smí technik ve schváleném servisu aktualizovat; vždy po záloze |
| `log_level` | `debug`, `info`, `warning`, `error`; tajné údaje nejsou v logu v žádné úrovni |

## Soubory v `/data`

Všechny s právy 600. Formát souboru s tokeny členů popisuje
`docs/uzivatele-json.md` v repozitáři agenta.
Povolení bezpečí (`bezpeci.json`), záznam „telefon byl doma“ (`doma_zarizeni.json`) a audit bezpečí
(`bezpeci_audit.jsonl`, 90 dní) popisuje `docs/bezpeci.md` v repozitáři agenta.
Token uživatele HA „Lussi servis“ (`servis_uzivatel.json`), začátky oken tunelu (`servis_tunel.json`)
a audit servisu (`servis_audit.jsonl`, 90 dní)
popisuje `docs/adr/0005-servis-provozovatele.md` v repozitáři agenta.

## Aktualizace

Agent jednou za hodinu (a hned, když cloud nové vydání ohlásí) zjistí v Lussi cloudu, jestli je pro dům nové
vydání, a stáhne ho do
`/share/lussi/aktualizace/k-instalaci/`. Nainstaluje ho dům sám v noci (2:00–4:30) po vlastní kontrole
podpisu a obsahu. Agent nic neinstaluje, kromě první instalace (níže). Výsledek instalace (`stav.json` od domu) posílá agent beze změny
do cloudu, aby provozovatel věděl, jak aktualizace dopadla; stejně hlásí i vlastní stav stahování. Stav
stahování je na panelu a v `/data/aktualizace_agent.json`
(`docs/adr/0006-doruceni-aktualizaci.md` v repozitáři agenta).

**První instalace Lussi core.** Když dům ještě nemá integraci Lussi core (`custom_components/naboo`), agent
po stažení prvního vydání ověří jeho podpis, obsah a verzi Home Assistantu, pro kterou je, rozbalí jen integraci do `/config/custom_components/naboo`,
restartuje Home Assistant a integraci přidá (bez tokenu TMDB, region CZ). Zbytek vydání pak nainstaluje Lussi
core sama hned, ne až v noci. Výsledek je v panelu a v `/share/lussi/aktualizace/prvni-instalace.json`. Když
integrace už je, agent nic nedělá; každé další vydání instaluje dům. Kvůli tomu má doplněk zápis do
konfigurace Home Assistantu, jinam než do `custom_components/naboo` ale nezapisuje
(`docs/adr/0008-prvni-instalace-lussi-core.md` v repozitáři agenta).

**Kanály a přechod mezi nimi.** Repozitář má dva doplňky: **Lussi agent** (stabilní) a **Lussi agent
(pilot)** pro zkušební domy, který dostává každou novou verzi hned. Nainstaluj jen jeden. Přechod bez nového
párování: nainstaluj druhý doplněk a nespouštěj ho, ve starém zapni volbu **Předat data druhému kanálu** a ulož.
Agent data zašifrovaně předá a zastaví se; pak spusť nový doplněk. Starý můžeš po ověření odinstalovat.

## Servis na dálku (K147)

Technik Lussi pomůže na dálku jen na žádost, kterou správce domácnosti schválí v aplikaci, a jen v jejím
časovém okně. Vidí stav zařízení a chyby, ne zámky, alarm, kamery ani osobní obsah. Zařízení ovládá, jen když
to správce povolil. Doma (8444) servis nefunguje. Správce vidí v aplikaci, co technik dělal.

## Bezpečí domu (E1+)

Vrata, bránu, dveře, okna, zámky a alarm ovládá dospělý člen z aplikace jen s povolením od správce
(výchozí: zapnuto jen správcům). Odemknout, otevřít a vypnout alarm jde jen po ověření otiskem nebo obličejem
v telefonu, které brána sama ověří podpisem. Na dálku jen z telefonu, který se už jednou připojil doma.
Dítě a host bezpečí nevidí. Správce vidí v aplikaci, kdo co kdy udělal.
