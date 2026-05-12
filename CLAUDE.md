# CLAUDE.md — Praktijktools

Werkgeheugen voor toekomstige Claude-sessies in deze repo. Lees dit *eerst* voordat je iets wijzigt.

## Wat dit is

Een persoonlijke spreekkamer-companion voor abdelkader (huisarts, Het Roosendael, Roermond). Doel: één plek waar hij op zijn iPhone razendsnel klinische scores, switches en wijzers opzoekt tijdens een consult. Werkt offline. Nu Progressive Web App (Add-to-Home-Screen op iOS); op termijn wikkelen tot native iOS-app voor de App Store.

De app is *geen* CE-gemerkt medisch hulpmiddel. Disclaimer staat in de footer en in elke README. Bij grenswaarden of complexe casuïstiek altijd NHG, KNMP-Kennisbank, ESC, SWAB of Pallialine raadplegen.

## Repo-layout

```
Praktijktools/
├── index.html              # legacy / landing — NIET het hoofdbestand
├── README.md               # spiegel van huisarts-tools/README.md
├── CLAUDE.md               # dit bestand
└── huisarts-tools/         # ← HIER zit de werkende PWA
    ├── index.html          # de hele app, één bestand
    ├── manifest.json       # PWA manifest
    ├── sw.js               # service worker (cache-versie hier verhogen bij updates)
    ├── icon-192.png · 512.png · maskable-512.png
    ├── vercel.json
    └── README.md
```

**Belangrijk:** `huisarts-tools/index.html` is het enige bestand dat je bewerkt voor functionaliteit. Alles zit erin: HTML, CSS, JS, alle tools.

Let op naam-casing: GitHub-repo en lokale map heten `Praktijktools` (hoofdletter P, correcte spelling). Vroeger stond hij als `prakijktools` (typo) — als je die nog tegenkomt, het is dezelfde repo onder de juiste naam.

## Architectuur — bewuste keuzes

- **Single-file** zodat abdelkader het in elke teksteditor kan openen en direct ziet wat hij doet. Geen `npm install`, geen Webpack, geen TypeScript-config die kapot kan.
- **Vanilla JS** met een mini App-object dat hash-based routing doet. Back-knop op iOS werkt daardoor native.
- **Service worker** met network-first voor HTML (zodat updates direct landen) en cache-first voor assets. Cache-naam staat in `sw.js` als `praktijktools-vN` — verhoog bij een release die je wilt forceren.
- **Clinical Modern design taal**: Geist Sans (Vercel) voor titels en body, JetBrains Mono voor numerieke output. Koel klinisch wit (`#FAFAFA`), één accent-kleur (`#0F766E` deep teal). Geen schaduwen-die-schreeuwen of gradients-die-flashen. Designstijl matcht de skills `design-taste-frontend` en `redesign-existing-projects`. Serif-fonts zijn expliciet verboden voor clinical UI volgens die skill.
- **iOS-first**: viewport-fit cover, safe-area insets, `user-scalable=no` om dubbeltap-zoom in inputs te voorkomen, `apple-mobile-web-app-capable` voor fullscreen na install. Search-bar is sticky bovenaan zodat ze altijd binnen handbereik blijft.

## Informatie-architectuur (v0.3 — herstructurering)

Vroeger acht categorieën, deels op tool-type (Rekenformules, Switch & rotatie), deels op orgaan (Cardiovasculair). Sinds v0.3 zeven categorieën, allemaal op **klinisch scenario** — de mentale stap die een arts zet bij een patiëntvraag:

1. **Acuut & spoed** — anafylaxie, hoofdtrauma, sepsis-triage, LRINEC (toekomst)
2. **Hart & vaten** — eGFR, CHA₂DS₂-VASc + HAS-BLED, SCORE2, Wells DVT
3. **Infectie & antibiotica** — Centor/McIsaac, antibiotica-keuzehulp per indicatie, paracetamol pediatrisch
4. **Psyche & GGZ** — PHQ-9, GAD-7, AUDIT-C, ADHD-medicatie
5. **Medicatie & rotatie** — antidepressiva-switch, opiatenrotatie, benzo-equivalenten, corticosteroïd-equivalenten, CYP-interacties, STOPP/START
6. **Endocrien & metabool** — FINDRISC, anticonceptie-keuzehulp, HRT, testosteron-suppletie
7. **Neuro & anatomie** — dermatomen + myotomen, hersenzenuwen I-XII

Bovenop deze categorisering staan **drie navigatie-bouwstenen** die de UI scanbaar houden:

- **Persistente zoekbalk** sticky bovenaan, filter zoekt op name + desc + keywords array
- **Favorieten-strip** (horizontaal scrollable, tot 8 pinned tools) — gebruiker tikt het ster-icoon naast de topbar-titel om vast te pinnen. Opslag in `localStorage` via `Favs` util-object.
- **Tags op cards** (`tag-acuut`, `tag-pediatrisch`, `tag-lookup`) — kleine pill onderaan een tool-card voor visuele context bij scannen.

## Tool-patroon

```javascript
TOOLS.push({
  id: 'egfr',                          // uniek; gebruikt in #/<id>
  name: 'eGFR + medicatie-flags',
  category: 'Hart & vaten',            // moet in CAT_ORDER staan
  tag: 'pediatrisch',                  // optioneel: acuut | pediatrisch | lookup
  metaLabel: 'CKD-EPI 2021',           // klein label boven kaart, monospace
  desc: 'Nierfunctie met automatische dosering-waarschuwingen',
  keywords: ['nierfunctie','egfr','creatinine','metformine','doac','ckd'],
  featured: true,                      // optioneel; grotere kaart met teal-tint op home
  render() {
    return `<div class="tool-view"> ... HTML met inputs en een #tool-out div ... </div>`;
  },
  onMount() {
    // Event-listeners aansluiten op de inputs uit render()
    computeEGFR();
  }
});

function computeEGFR() {
  // Lees inputs, bereken, schrijf naar #tool-out
}
```

### Categorie-iconen
SVG-paths staan inline in `const CATEGORY_ICONS = {...}` boven `renderShell`. Elke categorie heeft één 24×24 Phosphor-style outline-icoon, gerenderd in een 28px-tile met `--accent-bg` background. Nieuwe categorie toevoegen? Voeg path-string toe aan het object.

### Favorieten-API
```javascript
Favs.list()        // → ['egfr', 'adhd-meds', ...]
Favs.has(id)       // → boolean
Favs.toggle(id)    // → toggles + persisteert, returns nieuwe state
```
Cap is 8 items (oudste wordt afgesneden bij overschrijden). `App.toggleFav(id)` is de UI-handler die ook re-rendert.

## Helpers (hergebruiken, niet dupliceren)

| Helper          | Doel                                                        |
|-----------------|-------------------------------------------------------------|
| `$(id)`         | shorthand voor `document.getElementById`                    |
| `val(id)`       | leest waarde uit input/checkbox/number-veld                 |
| `fmt(n, dec)`   | getal → string met komma-scheidsteken en gewenste decimalen |
| `escape(s)`     | HTML-escape voor user input in templates                    |

### CSS-bouwstenen
- `.tool-view` — outer wrapper
- `.result`, `.result-eyebrow`, `.result-value`, `.result-interpretation` — outputblok met teal-tint en pulserende dot
- `.tag tag-red|blue|green|yellow|amber` — pill-badges binnen tool-content
- `.flag flag-red|...` — grotere kleurvlakken (medicatie-waarschuwingen)
- `.scale-options` — radio-grid (PHQ-9 / GAD-7 patroon, active = teal)
- `.checkrow` — rij met checkbox + label
- `.matrix-row` met `.matrix-from-to` + `.matrix-strategy` — lookup-rij voor tabellen (dermatomen, hersenzenuwen, antidepressiva-switch)
- `.tool-card-tag.tag-acuut|pediatrisch|lookup` — visuele tag op tool-card

### Referentie-tools om van te kopiëren
- **eGFR** (id `egfr`) — input + berekening + flags pattern
- **PHQ-9** (id `phq9`) — multi-item Likert scale met optelsom
- **Antibiotica-keuzehulp** (id `ab-keuze`) — select + segmented control + protocol-lookup
- **Dermatomen** (id `dermatomes`) — pure lookup-tabel zonder inputs

## Routing

`location.hash` is single source of truth. Leeg → `renderHome()` met search + favorites + categorieën. `#/<id>` → `tool.render() + onMount()`. Navigeren: `App.navigate('toolid')`. Terug: `App.back()` of native iOS-swipe.

## Lokaal draaien

```bash
cd huisarts-tools
python3 -m http.server 8000
# open http://localhost:8000 in Safari of Chrome
```

Service workers werken niet via `file://`, dus een echte HTTP-server is verplicht.

## Deployen

Repo is gekoppeld aan Vercel via GitHub. `git push origin main` triggert auto-deploy. Live op de Vercel-URL die je hebt geconfigureerd.

**Bij elke nieuwe release** waarvan je zeker wilt zijn dat hij overal landt: verhoog `CACHE` in `sw.js` (`praktijktools-vN` → `vN+1`), commit, push. De service worker pakt de nieuwe versie automatisch op bij volgende opening.

## Roadmap — wat er nog moet komen

Inventaris die nog te bouwen is, gemapt naar de nieuwe categorieën:

- *Acuut & spoed*: qSOFA / NEWS2, hoofdtrauma NICE-criteria, LRINEC (necrotiserende fasciitis)
- *Hart & vaten*: Cockcroft-Gault, ABCD² (TIA-risico)
- *Infectie & antibiotica*: uitgebreidere antibiotica-keuzehulp (sinusitis, prostatitis, mastitis), SWAB-empirisch
- *Psyche & GGZ*: EPDS (postpartum), MoCA-basis
- *Medicatie & rotatie*: Anticholinerge Belasting (ACB-score)
- *Endocrien & metabool*: schildklier-titratie levothyroxine, insuline-conversie
- *Neuro & anatomie*: Ottawa Ankle/Knee Rules, FRAX-NL (osteoporose 10-jaars)
- *Donderberg-specifiek*: tolk-/cultuurgevoelige consultatie-aide, Ramadan-medicatie

Patroon volgen — geen nieuwe architectuur introduceren tenzij abdelkader expliciet vraagt.

## Naar App Store (langere termijn)

Capacitor-wrapper is de pragmatische route: wikkel de PWA in een WKWebView, koppel native API's (notificaties, biometrie, deeplinks), bundel als iOS-app. Voorwaarden voor App Store-acceptatie: privacybeleid-URL, support-URL, screenshots in 6.7"/6.5"/5.5", en — voor medische apps — duidelijke disclaimer plus eventueel CE-traject als klinische beslissingen op de tools vertrouwen.

Eerste vraag bij die stap: wil abdelkader het puur voor zichzelf installeerbaar maken (Apple Developer-account, ad-hoc distributie) of breed verspreiden naar collega-huisartsen (TestFlight/App Store)? Bepaalt de eisenlat.

## Conventies bij wijzigingen

- **Nederlands** voor alle UI-tekst, commentaar in code mag Engels.
- **Geen frameworks toevoegen.** Geen React, geen Vue, geen build-step. Eén HTML-bestand.
- **Geen externe libraries** behalve Google Fonts (Geist + JetBrains Mono). Geen jQuery, geen Tailwind via CDN. Custom CSS volstaat.
- **Inline alles** — als een nieuwe tool een sliding panel of modal nodig heeft, bouw het inline in `index.html`, niet als losse component.
- **Numerieke output** altijd in monospace via `var(--font-mono)` of `.result-value` (die zit zelf op Geist met `font-variant-numeric: tabular-nums`).
- **Klinische uitkomsten** altijd in een `.result`-blok met eyebrow + value + interpretation, plus relevante flags als `.flag` of `.tag`.
- **Bronnen vermelden** bij elke tool die uit een richtlijn of validatiestudie komt — in de disclaimer onderaan de tool. NHG-Standaarden, SWAB, KNMP-Kennisbank, Kinderformularium, ESC, Akwa GGZ-richtlijnen zijn de Nederlandse standaard-referenties voor deze app.
- **Disclaimer** niet weghalen of verzwakken.
- **Medische correctheid** is non-negotiable. Bij twijfel: liever de bron citeren en de gebruiker laten opzoeken, dan een specifiek getal noemen dat fout is. Bij doseringen altijd dubbelchecken tegen Kinderformularium of FK.

## Persoonlijke context

- abdelkader is huisarts in Het Roosendael (Roermond), wijk Donderberg — achterstandswijk met onder andere veel Marokkaanse en Turkse patiënten, dus tolk-/cultuurgevoelige tools zijn relevant.
- Schrijfvoorkeur: narratief, beknopt, helder, geen bullets in lopende tekst, diepe analyse boven oppervlakkige opsommingen. Houd deze stijl ook aan in app-tekst (interpretaties, korte uitleg-blokjes).
- Werkt graag in single-file architectuur; vermijd over-engineering.

## Versie

- v0.1 — initiële release, 18 tools.
- v0.2 — redesign (Geist + teal), surfaces opwaardering.
- v0.3 — IA-herstructurering naar 7 klinische domeinen, favorieten-systeem, categorie-iconen, tags op cards, 6 nieuwe tools (ADHD-medicatie, antibiotica-keuzehulp, paracetamol pediatrisch, testosteron-suppletie, dermatomen, hersenzenuwen). Totaal 24 tools.

---

*Laatst bijgewerkt: 13 mei 2026.*
