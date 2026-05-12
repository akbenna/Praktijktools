# CLAUDE.md — Praktijktools

Werkgeheugen voor toekomstige Claude-sessies in deze repo. Lees dit *eerst* voordat je iets wijzigt.

## Wat dit is

Een persoonlijke spreekkamer-companion voor abdelkader (huisarts, Het Roosendael, Roermond). Doel: één plek waar hij op zijn iPhone razendsnel klinische scores, switches en wijzers opzoekt tijdens een consult. Werkt offline. Nu Progressive Web App (Add-to-Home-Screen op iOS); op termijn wikkelen tot native iOS-app voor de App Store.

De app is *geen* CE-gemerkt medisch hulpmiddel. Disclaimer staat in de footer en in elke README. Bij grenswaarden of complexe casuïstiek altijd NHG, KNMP-Kennisbank, ESC of Pallialine raadplegen.

## Repo-layout

```
prakijktools/
├── index.html              # legacy / landing — ~47k tokens; NIET het hoofdbestand
├── README.md               # spiegel van huisarts-tools/README.md
└── huisarts-tools/         # ← HIER zit de werkende PWA
    ├── index.html          # de hele app, één bestand
    ├── manifest.json       # PWA manifest
    ├── sw.js               # service worker (cache-versie hier verhogen bij updates)
    ├── icon-192.png
    ├── icon-512.png
    ├── icon-maskable-512.png
    ├── vercel.json
    └── README.md
```

**Belangrijk:** `huisarts-tools/index.html` is het enige bestand dat je bewerkt voor functionaliteit. Alles zit erin: HTML, CSS, JS, alle tools. Bewust geen build-step, geen npm, geen frameworks.

## Architectuur — bewuste keuzes

- **Single-file** zodat abdelkader het in elke teksteditor kan openen en direct ziet wat hij doet. Geen `npm install`, geen Webpack, geen TypeScript-config die kapot kan.
- **Vanilla JS** met een mini App-object dat hash-based routing doet. Back-knop op iOS werkt daardoor native.
- **Service worker** met network-first voor HTML (zodat updates direct landen) en cache-first voor assets. Cache-naam staat in `sw.js` als `praktijktools-v1` — verhoog dit bij een release die je wilt forceren.
- **Editorial Minimalist** design taal: Newsreader serif voor titels, system-sans voor body, JetBrains Mono voor getallen. Warme monochrome palette met `--bg #F7F6F3`. Geen schaduwen of gradients. Designstijl matcht de skill `minimalist-ui`.
- **iOS-first**: viewport-fit cover, safe-area insets, `user-scalable=no` om dubbeltap-zoom in inputs te voorkomen, `apple-mobile-web-app-capable` voor fullscreen na install.

## Tool-patroon

Elke tool is een object in de globale `TOOLS`-array. Patroon:

```javascript
TOOLS.push({
  id: 'egfr',                          // uniek; gebruikt in #/<id>
  name: 'eGFR + medicatie-flags',
  category: 'Rekenformules',           // moet in CAT_ORDER staan
  metaLabel: 'CKD-EPI 2021',           // klein label boven kaart
  desc: 'Nierfunctie met automatische dosering-waarschuwingen',
  keywords: ['nierfunctie','egfr','creatinine','metformine','doac','ckd'],
  featured: true,                      // optioneel; grotere kaart op home
  render() {
    return `<div class="tool-view"> ... HTML met inputs en een #tool-out div ... </div>`;
  },
  onMount() {
    // Event-listeners aansluiten op de inputs uit render()
    // Bijv. $('age').addEventListener('input', computeEGFR);
    computeEGFR();
  }
});

function computeEGFR() {
  // Lees inputs, bereken, schrijf naar #tool-out
}
```

### Verplichte velden
`id`, `name`, `category`, `desc`, `keywords`, `render()`.

### Optionele velden
`metaLabel`, `featured`, `onMount()`.

### Categorieën — `CAT_ORDER` (volgorde op homepage)

1. Rekenformules
2. Cardiovasculair
3. GGZ-scores
4. Switch & rotatie
5. Pharmacowijzer
6. Polyfarmacie
7. Gynaecologie & endocrien
8. Acuut

Nieuwe categorie? Voeg toe aan `CAT_ORDER` (vroeg in het script-blok). Houd de lijst kort; een tool die er net niet in past hoort meestal beter onder een bestaande categorie.

## Helpers (hergebruiken, niet dupliceren)

| Helper          | Doel                                                        |
|-----------------|-------------------------------------------------------------|
| `$(id)`         | shorthand voor `document.getElementById`                    |
| `val(id)`       | leest waarde uit input/checkbox/number-veld                 |
| `fmt(n, dec)`   | getal → string met komma-scheidsteken en gewenste decimalen |
| `escape(s)`     | HTML-escape voor user input in templates                    |

### CSS-bouwstenen
- `.tool-view` — outer wrapper
- `.result`, `.result-eyebrow`, `.result-value`, `.result-interpretation` — outputblok
- `.tag tag-red|blue|green|yellow|amber` — pill-badges
- `.flag flag-red|...` — grotere kleurvlakken (bijv. medicatie-waarschuwingen)
- `.scale-options` — radio-grid (PHQ-9 patroon)
- `.checkrow` — rij met checkbox + label

### Twee referentie-tools om van te kopiëren
- **eGFR** — input + berekening + flags pattern. Zoek `TOOLS.push({ id: 'egfr'`.
- **PHQ-9** — multi-item Likert scale met optelsom. Zoek `TOOLS.push({ id: 'phq9'`.

Voor een nieuwe rekenformule volg eGFR. Voor een nieuwe vragenlijst volg PHQ-9.

## Routing

`location.hash` is de single source of truth. `App.handleRoute()` (regel ~699) leest hem:
- Leeg → `renderHome()` met zoekbalk en gegroepeerde categorieën
- `#/<id>` → tool gevonden in `TOOLS` → `render()` + `onMount()`

Navigeren: `App.navigate('toolid')`. Terug: `App.back()` of native iOS-swipe.

## Zoeken

`renderToolSections(search)` filtert op:
- `name` (case-insensitive substring)
- `desc` (case-insensitive substring)
- `keywords` array (exact match per term)

Bij actieve zoekopdracht wordt de groepering uitgezet en een platte "Resultaten"-lijst getoond. Keystroke-debounce is *niet* ingebouwd — het lijstje is klein genoeg dat het niet nodig is. Voeg het pas toe als de TOOLS-array > 60 items wordt.

## Lokaal draaien

```bash
cd huisarts-tools
python3 -m http.server 8000
# open http://localhost:8000 in Safari of Chrome
```

Service workers werken niet via `file://`, dus een echte HTTP-server is verplicht.

## Deployen

```bash
cd huisarts-tools
vercel --prod
```

Live op `praktijktools.vercel.app` (of het gekoppelde subdomein). Custom domein staat in Vercel → Settings → Domains; CNAME-record nodig bij de DNS-provider.

**Bij elke nieuwe release** waarvan je zeker wilt zijn dat hij overal landt:
1. Verhoog `CACHE` in `sw.js` (`praktijktools-v1` → `praktijktools-v2`).
2. `vercel --prod`.
3. Bij volgende opening van de app pakt de service worker de nieuwe versie automatisch.

## iPhone-installatie (zonder App Store)

1. App-URL openen in **Safari** op iPhone (werkt niet in Chrome iOS — Apple-restrictie).
2. Deelknop (vierkant + pijl omhoog).
3. "Zet op beginscherm" / "Add to Home Screen".

App draait fullscreen, eigen splash screen, en werkt offline zodra cache gevuld is.

## Roadmap — wat er nog moet komen

Tools die op de inventaris stonden maar nog te bouwen:

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

Patroon volgen — geen nieuwe architectuur introduceren tenzij abdelkader expliciet vraagt.

## Verder denken — naar App Store

Het pad van PWA naar App Store loopt typisch via een dunne native wrapper:
- **Capacitor** (Ionic, gratis) — wikkelt de PWA in een WKWebView, geeft toegang tot native API's (notificaties, biometrie, deeplinks). Geschikt als de app puur lokaal blijft en de waarde in offline klinische tools zit.
- **PWABuilder** (Microsoft) — automatische iOS-bundle uit een bestaande PWA, soms net iets minder controle.

App Store-acceptatie vraagt: privacybeleid-URL, support-URL, screenshots voor 6.7"/6.5"/5.5", en — voor medische apps — duidelijke disclaimer plus eventueel CE-traject als er klinische beslissingen op vertrouwd moeten worden.

Eerste vraag bij die stap: wil abdelkader het puur voor zichzelf installeerbaar maken (Apple Developer-account, ad-hoc distributie) of breed verspreiden naar collega-huisartsen (TestFlight/App Store)? Bepaalt de eisenlat.

## Conventies bij wijzigingen

- **Nederlands** voor alle UI-tekst, commentaar in code mag Engels.
- **Geen frameworks toevoegen.** Geen React, geen Vue, geen build-step. Eén HTML-bestand.
- **Geen externe libraries** behalve Google Fonts. Geen jQuery, geen Tailwind via CDN. Custom CSS volstaat.
- **Inline alles** — als een nieuwe tool een sliding panel of modal nodig heeft, bouw het inline in `index.html`, niet als losse component.
- **Numerieke output** altijd in monospace (`var(--font-mono)`).
- **Klinische uitkomsten** altijd in een `.result`-blok met eyebrow + value + interpretation, plus relevante flags als `.flag` of `.tag`.
- **Bronnen vermelden** bij elke tool die uit een richtlijn of validatiestudie komt — kleine `data-source` regel onderaan de tool, monospace.
- **Disclaimer** niet weghalen of verzwakken.

## Persoonlijke context

- abdelkader is huisarts in Het Roosendael (Roermond), wijk Donderberg — achterstandswijk met o.a. veel Marokkaanse en Turkse patiënten, dus tolk-/cultuurgevoelige tools zijn relevant.
- Schrijfvoorkeur: narratief, beknopt, helder, geen bullets in lopende tekst, diepe analyse boven oppervlakkige opsommingen. Houd deze stijl ook aan in app-tekst (interpretaties, korte uitleg-blokjes).
- Werkt graag in single-file architectuur; vermijd over-engineering.

## Versie

v0.1 — initiële release, 18 tools live.

---

*Laatst bijgewerkt: 12 mei 2026.*
