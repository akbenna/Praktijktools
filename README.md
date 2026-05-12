# Praktijktools

Een Progressive Web App (PWA) met klinische rekentools, switch-kaarten en wijzers voor de Nederlandse huisartsenpraktijk. Werkt offline. Installeerbaar op iPhone zonder App Store.

## Wat zit erin (v0.1)

Achttien gevalideerde tools, verdeeld over zeven domeinen:

- **Rekenformules** — eGFR (CKD-EPI 2021), Centor/McIsaac, Wells DVT, FINDRISC
- **Cardiovasculair** — CHA₂DS₂-VASc + HAS-BLED, SCORE2 + CV-leeftijd
- **GGZ-scores** — PHQ-9, GAD-7, AUDIT-C
- **Switch & rotatie** — antidepressiva-switch, opiatenrotatie, benzo-equivalenten + afbouw, corticosteroïd-equivalenten
- **Pharmacowijzer** — CYP-interacties met SIP-score
- **Polyfarmacie** — STOPP/START quick screener
- **Gynaecologie & endocrien** — anticonceptie-keuzehulp, HRT-stroomschema
- **Acuut** — anafylaxie-protocol met doseringsschema's

## Architectuur

Bewust simpel:

- Eén HTML-bestand met embedded CSS en JS. Geen frameworks, geen build-step, geen npm.
- Service worker voor offline cache. Werkt zonder wifi in de spreekkamer.
- PWA-manifest voor installatie op het beginscherm.
- Hash-based routing (`#/toolid`) zodat de back-knop op iOS gewoon werkt.

Wijzigingen aanbrengen kan in elke teksteditor. Geen `npm install`. Open `index.html` lokaal in de browser en je ziet meteen wat je doet.

## Lokaal testen

```bash
cd huisarts-tools
python3 -m http.server 8000
# open http://localhost:8000 in Safari of Chrome
```

(Een echte HTTP-server is nodig omdat service workers niet werken via `file://`.)

## Deployment op Vercel

Aanname: je hebt een Vercel-account met de CLI geïnstalleerd (`npm i -g vercel`).

```bash
cd huisarts-tools
vercel
```

Volg de prompts. Bij de eerste keer kies je een naam (bv. `praktijktools`) en linkt aan een team. Vercel deployed automatisch en geeft je een URL als `praktijktools.vercel.app`.

Voor een eigen subdomein (bijvoorbeeld `tools.akbennaghmouch.nl` of een subdomein onder `provita.care`):

1. Ga naar de Vercel-dashboard van het project → **Settings** → **Domains**.
2. Voeg je gewenste domein toe.
3. Vercel toont een CNAME-record dat je bij je DNS-provider moet toevoegen.

Daarna is de app beschikbaar op je eigen URL — vereist voor PWA-installatie (HTTPS verplicht).

## Installeren op iPhone (zonder App Store)

1. Open de app-URL in **Safari** op je iPhone. (Werkt niet in Chrome op iOS — Apple staat alleen Safari toe voor "Add to Home Screen".)
2. Tik op het **deelknop**-icoon (vierkant met pijl omhoog) onderaan het scherm.
3. Scrol naar beneden en tik op **"Zet op beginscherm"** of **"Add to Home Screen"**.
4. Bevestig de naam ("Praktijktools") en tik op **Toevoegen**.

Het icoon verschijnt op je beginscherm. Bij openen draait de app fullscreen, zonder Safari-balk, met eigen splash screen. Vanaf dat moment werkt het ook offline — alle tools zijn lokaal gecached.

## Installeren op Android

Op Android Chrome krijg je automatisch een banner met "App installeren", of via het menu (drie puntjes) → **App installeren**. Werkt vergelijkbaar.

## Updaten van de app

Pas `index.html` aan, commit, deploy opnieuw (`vercel --prod`). De service worker controleert bij elke navigatie of er een nieuwe versie is en haalt deze op de achtergrond op. Bij volgende opening zie je de update.

Als je expliciet een nieuwe versie wilt forceren: verhoog het `CACHE` versienummer in `sw.js` (bv. van `praktijktools-v1` naar `v2`). Dat invalideert de oude cache.

## Wat ontbreekt nog (roadmap)

Tools die op de inventaris stonden maar nog niet zijn geïmplementeerd:

- ABCD² (TIA-risico)
- Ottawa Ankle/Knee Rules
- FRAX-NL (osteoporose 10-jaars risico)
- qSOFA / NEWS2 (sepsis-triage)
- EPDS (postpartum depressie)
- MoCA-basis
- Anticholinerge belasting (ACB-score)
- Cockcroft-Gault
- SWAB-eerstelijns wijzer per indicatie
- Insuline-conversie
- Testosteronsubstitutie
- Schildklier-titratie levothyroxine
- Hoofdtrauma NICE-criteria
- LRINEC (necrotiserende fasciitis)
- Donderberg-specifiek: tolk-/cultuurgevoelige consultatie-aide, Ramadan-medicatie

Elk daarvan is een uitbreiding in `index.html`: een `TOOLS.push({...})`-blok toevoegen en eventuele helperfunctie eronder. Het stramien staat — bijbouwen is een kwestie van patroon volgen.

## Disclaimer

Deze app is een hulpmiddel voor de zorgprofessional, geen vervanging van klinisch oordeel. Numerieke uitkomsten zijn schattingen op basis van gevalideerde modellen; bij grenswaarden of complexe casuïstiek altijd bronnenmateriaal raadplegen (NHG-Standaarden, KNMP-Kennisbank, ESC-richtlijnen, Pallialine).

Niet gevalideerd als CE-gemerkt medisch hulpmiddel. Bij gebruik in een PlusPraktijk- of zorggroepsetting eerst lokale privacy-impactanalyse.

## Versie

v0.1 — initiële release. Voor: Het Roosendael, Roermond.
