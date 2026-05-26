# Pool-varmepumpe styret af spotpriser - Home Assistant setup

Komplet setup til at styre din Duratech Dura+ 28Ti via IntelliPool's AUX-indgang og en Shelly, baseret på spotpriser fra Andel/Nord Pool. Designet til at køre på en **Mac mini** med Docker.

**Hvad systemet gør:** Henter spotpriser fra Energi Data Service hver time, lægger Cerius nettarif, elafgift og moms ovenpå, finder de N billigste timer i dit ønskede tidsvindue (default 10-17, undgår spidslast 17-21), og lukker/åbner Shelly-relæet derefter. Shelly trigger AUX-indgangen på IntelliPool, som så styrer varmepumpen blødt og sikkert.

---

## Indholdsfortegnelse

1. [Hvad du skal bruge](#1-hvad-du-skal-bruge)
2. [Forberedelse af Mac mini](#2-forberedelse-af-mac-mini)
3. [Hent dette projekt](#3-hent-dette-projekt)
4. [Start Home Assistant første gang](#4-start-home-assistant-første-gang)
5. [Installer HACS (community store)](#5-installer-hacs-community-store)
6. [Installer Energi Data Service integrationen](#6-installer-energi-data-service-integrationen)
7. [Sæt Shelly op](#7-sæt-shelly-op)
8. [Aktiver iPhone-appen og notifikationer](#8-aktiver-iphone-appen-og-notifikationer)
9. [Test at det hele virker](#9-test-at-det-hele-virker)
10. [Justering og fejlfinding](#10-justering-og-fejlfinding)

---

## 1. Hvad du skal bruge

- **Mac mini** (eller anden Mac der står tændt 24/7)
- **iPhone** med App Store
- **Shelly relæ** (du har én - vi konfigurerer den i trin 7)
- **WiFi** som både Mac mini og Shelly er på (samme netværk)
- **IntelliPool** med AUX-indgang konfigureret til at trigge varmepumpe-scenarie (det aftaler vi separat med den manual du har)
- **Cirka 1-2 timer** til det første setup

Du skal IKKE kunne programmere. Alt foregår via web-interface og kommandoer du kan kopiere.

---

## 2. Forberedelse af Mac mini

### 2.1 Installer Docker Desktop

1. Åbn Safari, gå til <https://www.docker.com/products/docker-desktop/>
2. Klik **"Download for Mac"** og vælg den der passer til din Mac (Apple Silicon for M1/M2/M3, Intel for ældre Mac mini)
3. Åbn den downloadede `.dmg` og træk **Docker** ind i **Programmer**-mappen
4. Start **Docker** fra Launchpad
5. Accepter brugerbetingelserne. Du kan vælge "Use recommended settings"
6. Når Docker er startet, ser du en lille hval-ikon øverst i menulinjen. Når den ikke længere bevæger sig, er Docker klar
7. Du kan godt lukke Docker-vinduet - programmet kører videre i baggrunden

### 2.2 Installer Git (hvis det ikke allerede er der)

1. Åbn **Terminal** (find den i Launchpad eller spotlight med Cmd+Space → "Terminal")
2. Skriv: `git --version` og tryk Enter
3. Hvis det viser en versionsnummer er du klar. Hvis det spørger om at installere "Xcode Command Line Tools", så accepter og vent (5-10 min)

---

## 3. Hent dette projekt

I Terminal, skriv kommandoerne her - én linje ad gangen, tryk Enter efter hver:

```bash
cd ~
git clone https://github.com/hylle9/hello-world.git pool-automation
cd pool-automation
git checkout claude/iphone-pool-heater-automation-apPxn
```

Nu ligger alle filerne i `/Users/dit-brugernavn/pool-automation`. Du kan se dem i Finder ved at trykke `Cmd+Shift+G` og skrive `~/pool-automation`.

---

## 4. Start Home Assistant første gang

### 4.1 Start containeren

I Terminal (stadig i `pool-automation` mappen):

```bash
docker compose up -d
```

Det downloader Home Assistant (kan tage 5-15 minutter første gang) og starter den i baggrunden. Når den er færdig ser du `Container homeassistant Started`.

### 4.2 Åbn web-interfacet

I Safari, åbn: **<http://localhost:8123>**

Du ser "Preparing Home Assistant" - vent et minuts tid og refresh siden hvis intet sker.

### 4.3 Opret bruger og lokation

1. Vælg navn, brugernavn og password. **Skriv password ned** - du skal bruge det igen
2. Sæt **lokation** (vigtigt for solnedgang/op til pool-automatisering): tryk "Detect" eller indtast adresse
3. Sæt **tidszone** til Europe/Copenhagen, **enheder** til Metric, **valuta** til DKK
4. Klik gennem de sidste skærme. Du kan godt sige nej til alle de foreslåede integrationer - vi tilføjer dem selv

Du er nu i Home Assistant. Yay!

---

## 5. Installer HACS (community store)

HACS = Home Assistant Community Store. Det giver adgang til den danske Energi Data Service-integration som ikke er indbygget i HA som standard.

### 5.1 Kør installations-scriptet

I Terminal:

```bash
docker exec -it homeassistant bash -c "wget -O - https://get.hacs.xyz | bash -"
```

Du ser noget output der slutter med "Installation complete".

### 5.2 Genstart Home Assistant

```bash
docker restart homeassistant
```

Vent et minut, og refresh `http://localhost:8123` i Safari.

### 5.3 Aktiver HACS

1. I HA's venstre menu, gå til **Indstillinger** → **Enheder og tjenester**
2. Klik **+ TILFØJ INTEGRATION** nederst til højre
3. Søg efter **HACS** og klik på den
4. Sæt flueben i alle 4 checkbokse og klik **SEND**
5. Du får en **kode** og et **link**. Klik linket, log ind på GitHub (opret konto hvis du ikke har), indtast koden
6. Tilbage i HA: vent et øjeblik. HACS skulle nu være installeret
7. Du ser nu **HACS** i venstre menu

---

## 6. Installer Energi Data Service integrationen

### 6.1 Installer via HACS

1. Klik **HACS** i venstre menu
2. Søg **"Energi Data Service"** i søgefeltet
3. Klik på integrationen og klik **DOWNLOAD** nederst til højre
4. Genstart Home Assistant: **Indstillinger** → **System** → øverst til højre **Genstart** → **Genstart Home Assistant**
5. Vent et minut, log ind igen

### 6.2 Tilføj sensoren

1. **Indstillinger** → **Enheder og tjenester** → **+ TILFØJ INTEGRATION**
2. Søg **"Energi Data Service"**
3. Klik den - en dialog åbner
4. Sæt:
   - **Område**: `DK2` (Sjælland)
   - **Valuta**: `DKK`
   - **Inkluder VAT**: ✓ (sæt flueben)
   - **Brugerdefineret enhedsnavn**: lad stå tom (så hedder den `sensor.energi_data_service`)
5. Klik **SEND**

I næste skærm bliver du spurgt om **tariffer**:

6. **Inkluder tariffer**: ✓
7. **Netoperatør**: vælg **Cerius A/S**
8. **Tarif type**: vælg **C** (det er det almindelige hus-tarif under 25.000 kWh/år)
9. **Statslige tariffer**: ✓ (elafgift)
10. Klik **SEND**

Færdig! Du har nu en sensor der hedder `sensor.energi_data_service` som henter spotpris + nettarif + afgift + moms automatisk.

### 6.3 Tjek at det virker

1. **Udviklerværktøjer** (hammer-ikon i menuen, kan være under "≡" øverst til venstre) → fanen **Tilstande**
2. Skriv `sensor.energi_data_service` i søgefeltet
3. Du burde se en pris i kr/kWh og under attributter en lang liste med dagens timepriser

Hvis ikke - se [fejlfinding](#10-justering-og-fejlfinding).

---

## 7. Sæt Shelly op

### 7.1 Find Shelly'ens IP-adresse

1. Hvis Shelly'en endnu ikke er forbundet til WiFi:
   - Sæt strøm til den. Den opretter et WiFi-netværk med navn som `shelly1-XXXXXX`
   - På iPhone, gå i Indstillinger → WiFi → forbind til det netværk
   - Åbn Safari, gå til `http://192.168.33.1`
   - Under "Internet & Security" → "WiFi mode - Client" sæt den til dit hjemme-WiFi
   - Den genstarter og forbinder til dit WiFi
2. Find dens IP på dit netværk:
   - Åbn din router (typisk `http://192.168.1.1` eller `http://192.168.0.1`) → kig efter "Connected devices" eller "DHCP leases" → find `shelly...`
   - **Skriv IP'en ned** (fx `192.168.1.42`)

### 7.2 Tilføj Shelly i Home Assistant

1. I HA: **Indstillinger** → **Enheder og tjenester** → **+ TILFØJ INTEGRATION**
2. Søg **Shelly** og klik den
3. Indtast IP-adressen du noterede
4. Klik **SEND**

Shelly'en er nu tilføjet. Den vises som en eller flere enheder afhængigt af model.

### 7.3 Omdøb relæ-enheden

For at vores automatisering virker uden ændringer, skal switchen hedde præcis det rigtige:

1. **Indstillinger** → **Enheder og tjenester** → **Enheder** → find din Shelly
2. Klik på den. Du ser én eller flere "kontakter" (switches)
3. Klik på den relevante kontakt (den der skal styre AUX-indgangen)
4. Klik tandhjul-ikonet (⚙) øverst til højre
5. Sæt **Entitets-ID** til præcis: `switch.shelly_pool_aux`
6. Klik **OPDATER**

### 7.4 Konfigurer Shelly til tørkontakt (vigtigt!)

IntelliPool's AUX-indgang skal have **tørkontakt** (potentialefri), ikke 230V. På de fleste Shelly Plus 1 / Shelly 1 kan du:

1. Åbn Shelly'ens eget webinterface (skriv dens IP i Safari)
2. Find **Settings** → **Input/Output settings** eller **Relay settings**
3. Hvis muligt, sæt den til "Dry contact" mode (ingen forbindelse mellem L og O på outputs)

Den præcise procedure varierer pr. Shelly-model. Hvis du er i tvivl: skriv hvilken Shelly-model du har, så hjælper jeg specifikt.

---

## 8. Aktiver iPhone-appen og notifikationer

### 8.1 Hent app'en

1. På iPhone, åbn App Store
2. Søg **"Home Assistant"** (af Home Assistant)
3. Hent og åbn

### 8.2 Forbind til din HA

1. App'en burde finde din HA automatisk hvis iPhone er på samme WiFi
2. Hvis ikke: indtast manuelt `http://[MAC-MINI-IP]:8123` (find Mac mini'ens IP i Systemindstillinger → Netværk)
3. Log ind med samme bruger som i trin 4.3
4. Accepter at modtage notifikationer

### 8.3 Tilpas notifikationsnavnet i automatiseringen

Når du har logget ind i appen på iPhonen, opretter HA automatisk en notifikations-service med navn som `notify.mobile_app_din_iphones_navn`.

1. **Udviklerværktøjer** → **Tjenester**
2. I dropdown'en, søg efter `notify.mobile_app` - du ser den fulde service-ID
3. Skriv den ned

4. Åbn `homeassistant/automations.yaml` (i Finder under `~/pool-automation/homeassistant/`)
5. Find alle steder hvor der står `notify.mobile_app_din_iphone` og skift til det rigtige navn fra trin 3
6. Gem filen
7. I HA: **Udviklerværktøjer** → **YAML** → klik **AUTOMATISERINGER** for at genindlæse

---

## 9. Test at det hele virker

### 9.1 Aktiver pool-automatiseringen

1. **Indstillinger** → **Enheder og tjenester** → **Hjælpere** (Helpers)
2. Find `Pool-automatisering aktiv` og slå den **TIL**

### 9.2 Tjek hvad systemet tænker lige nu

1. **Udviklerværktøjer** → **Tilstande**
2. Søg `binary_sensor.pool_skal_varme_nu`
3. Du ser `on` eller `off` afhængig af om vi er i en billig time lige nu
4. Under attributter ser du `billigste_timer` - listen over de timer i dag hvor systemet vil varme

### 9.3 Manuel test af Shelly

1. **Oversigt** (Lovelace) → find `switch.shelly_pool_aux`
2. Tryk på den - du burde høre et klik fra Shelly'en
3. IntelliPool burde reagere på AUX-input (hvis kabling er på plads). Tjek IntelliPool's status

### 9.4 Tjek logfilen for fejl

```bash
docker logs --tail 100 homeassistant
```

Eller i UI: **Indstillinger** → **System** → **Logs**.

---

## 10. Justering og fejlfinding

### 10.1 Justér antal timer der varmes

I HA: **Indstillinger** → **Enheder og tjenester** → **Hjælpere**, find:
- `Antal billige timer poolen skal varme` (default 5)
- `Starttime for varmevindue` (default 10)
- `Sluttime for varmevindue` (default 17, eksklusiv - betyder sidste time er 16-17)

Ændringer træder i kraft øjeblikkeligt.

### 10.2 Pause hele automatiseringen midlertidigt

Slå `Pool-automatisering aktiv` fra i Hjælpere. Den næste pristjek vil ikke tænde Shelly'en.

### 10.3 Almindelige fejl

| Symptom | Årsag | Løsning |
|---|---|---|
| `sensor.energi_data_service` har ingen pris | Integrationen ikke konfigureret | Tjek trin 6, og at Mac mini har internet |
| `binary_sensor.pool_skal_varme_nu` viser altid `off` | `pool_automation_aktiv` er off | Tænd den i Hjælpere |
| Shelly skifter ikke | Forkert IP eller offline | Tjek Shelly'ens webinterface direkte i Safari |
| Notifikationer kommer ikke | Forkert `notify.`-navn i automatiseringen | Se trin 8.3 |
| Webinterface viser fejl efter restart | YAML-fejl i config | Kør `docker logs homeassistant` og kig efter "Invalid config" |

### 10.4 Opdater Home Assistant senere

```bash
cd ~/pool-automation
docker compose pull
docker compose up -d
```

### 10.5 Hvis alt går galt - start forfra

```bash
cd ~/pool-automation
docker compose down
rm -rf homeassistant/.storage homeassistant/home-assistant_v2.db*
docker compose up -d
```

ADVARSEL: Det sletter al din historik og konfiguration. Filerne `configuration.yaml`, `automations.yaml` mv. bevares.

---

## Hvad sker der videre?

Næste skridt vi skal aftale separat:

1. **AUX-indgangen på IntelliPool**: Find frem til den manual-side du nævnte, og afgør præcis hvordan vi kabler Shelly'en til at trigge det rigtige scenarie
2. **Tilpas tidsvindue**: 10-17 er sikkert for Cerius (undgår spidslast 17-21), men i højsommer hvor solen står højt 10-15 kunne det være endnu bedre
3. **Tilføj poolens vandtemperatur-sensor** så systemet kan stoppe når vandet er varmt nok (kræver enten en Shelly Plus Add-on med temperatursensor eller en MQTT-bro til IntelliPool)
4. **Dashboard på iPhone**: Sæt et flot dashboard op med pris-graf, dagens plan, manuel override-knap mv.
