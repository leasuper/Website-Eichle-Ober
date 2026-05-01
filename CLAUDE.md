# CLAUDE.md – Eichle Ober Website

Vereins-Website des Guggitaler Jass-Klubs. Statische HTML-Dateien, kein Build-Step.

---

## Stack & Hosting

| Was | Wie |
|-----|-----|
| Hosting | GitHub Pages → `main` branch |
| Domain | **eichleober.ch** (CNAME-Datei vorhanden) |
| Repo | https://github.com/leasuper/Website-Eichle-Ober |
| HTML/CSS | Statisch, kein Bundler, kein npm |
| React | 18.3.1 via CDN/UMD (JSX in `<script type="text/babel">`) |
| Babel | Standalone 7.29.0 via CDN (Browser-Transpiling) |
| Firebase | 10.12.0 Compat-SDK via CDN |

**Regel: Nie einen Build-Step einführen.** Alles muss als einzelne HTML-Datei im Browser laufen.

---

## Firebase-Konfiguration

```js
firebase.initializeApp({
  apiKey:      "AIzaSyAT95KlfXEiKAtRQ1PtMqctBOU-c5MW73A",
  authDomain:  "website-eichle-ober-bf1e5.firebaseapp.com",
  databaseURL: "https://website-eichle-ober-bf1e5-default-rtdb.europe-west1.firebasedatabase.app",
  projectId:   "website-eichle-ober-bf1e5",
  appId:       "1:405255994774:web:5c590b8dd16ce9ebfd31b7"
});
```

Der API-Key ist im HTML eingebettet — das ist so gewollt (Firebase-Client-Key, kein Server-Secret).

---

## Dateistruktur

```
eichle ober/
├── index.html          ← Hauptseite: Auth (Whitelist), Diskussion, Jasstafel-Wins
├── jass.html           ← Guggitaler Jass: Multiplayer-Kartenspiel (React)
├── jasstafel.html      ← Punktetafel-Tool
├── style.css           ← Globale Styles (kaum genutzt, alles inline)
├── CNAME               ← eichleober.ch
├── img/
│   ├── CH-Jasskartenset/   ← 36 Karten-PNGs: {suit}_{rank}.png
│   └── eichel-ober.gif
└── index_alt.html / NeuDesign2904.html  ← Entwürfe, nicht aktiv
```

---

## index.html

### Auth
- Google Sign-In via Firebase Auth (`signInWithPopup`)
- **Whitelist** (Allowlist): Erlaubte E-Mails stehen im `ALLOWED`-Array direkt in `index.html`
- `currentUser` ist eine globale Variable (kein React), wird in `onAuthStateChanged` gesetzt

### Design
- Dark Editorial: bg `#0b0b09`, ink `#f0ece0`, gold `#f0a830`, line `rgba(240,236,224,.1)`
- Fonts: **Syne** (Überschriften) + **DM Mono** (Mono-Akzente)
- Passwort-Overlay (`#pw-box`) deckt die Seite ab bis Auth erfolgreich

### Features
1. **Diskussionsplattform** – Posts + Kommentare in `diskussion/` (Firebase RTDB)
   - Jeder Post hat: `text`, `author`, `authorUid`, `timestamp`, `likes: []`, `dislikes: []`
   - Jeder Kommentar hat: `text`, `author`, `authorUid`, `timestamp`
   - Like/Dislike: Arrays von UIDs — gegenseitig ausschliessend
   - Edit-Button nur wenn `currentUser.uid === post.authorUid`

2. **Wins-Tracking** – Jasstafel-Gewinner in `wins/` (Firebase RTDB)

3. **CTA-Button** – "🃏 Als Gast Jassen" unter dem Google-Login, verlinkt auf `jass.html`

### Firebase RTDB-Pfade (index.html)
```
diskussion/{pushId}/
  text, author, authorUid, timestamp, likes[], dislikes[]
  kommentare/{pushId}/
    text, author, authorUid, timestamp
wins/{pushId}/
  name, timestamp
```

---

## jass.html — Guggitaler Jass

### Auth / Login
- **Kein Google-Login** — Gäste geben nur ihren Namen ein
- Name + generierte UID werden in `localStorage` gespeichert (`jass_uid`, `jass_name`)
- Beim nächsten Besuch: direkt in Lobby (kein Name-Input nötig)
- Kein Firebase Auth — kein `firebase-auth-compat.js` Script

### Spielprinzip
Kein Trumpf · Farbzwang · Wer am wenigsten Punkte hat, gewinnt (umgekehrter Jass).

### 6 Runden (rndIdx 0–5)
| # | Name          | Strafe                         |
|---|---------------|--------------------------------|
| 0 | Stiche        | +5 pro Stich                   |
| 1 | Schilten      | +10 pro Schiltenkarte          |
| 2 | Könige        | +20 pro König                  |
| 3 | Eichel Ober   | +40 wenn Eichel Ober gefangen  |
| 4 | Letzter Stich | +50 für letzten Stich          |
| 5 | Alles!        | alle obigen gleichzeitig       |

### State Machine (`G.phase`)
```
'waiting' → Lobby/Warteraum
'playing' → Karte wählen
'trick_result' → 1.5s Pause nach Stich
'round_result' → Rundenende-Anzeige
'game_over' → Endresultat
```

### Wichtige State-Felder
```js
{
  phase, rndIdx,                  // Phase & aktive Runde
  numPlayers,                     // 4 | 5 | 6
  seatNames: ['Name', ...],       // Index = Seat
  humanSeats: [0, 2],             // Welche Seats sind Menschen (rest = KI)
  trick: [{player, card}],        // Laufender Stich
  tricksDone,                     // 0–9 (je nach numPlayers)
  tricksWon: [[cards], ...],      // Gewonnene Stiche je Seat
  lastW, lead, cur,               // Stichgewinner / Anspiel / Aktueller
  totals: [0,0,0,0],             // Kumulierte Punkte
  history: [],                    // Scores pro Runde
  trickRes, rndRes,               // Anzeige-Objekte
  handSizes: [9,9,9,9]           // Kartenanzahl pro Seat (für CardBack)
}
```

### Kartenrepräsentation
```js
{ suit: 'Eichel'|'Schellen'|'Schilten'|'Rosen', rank: '6'|'7'|'8'|'9'|'B'|'U'|'O'|'K'|'A', id: 'Eichel-O' }
// RANKS = ['6','7','8','9','B','U','O','K','A']  (aufsteigend, ri() = Index)
// Bild: img/CH-Jasskartenset/${SUIT_FILE[suit]}_${RANK_FILE[rank]}.png
const SUIT_FILE = { Eichel:'eichel', Schellen:'schelle', Schilten:'schilte', Rosen:'rose' };
const RANK_FILE = { '6':'6','7':'7','8':'8','9':'9','B':'10','U':'under','O':'ober','K':'koenig','A':'ass' };
```

### Firebase RTDB-Struktur (jass.html)
```
jass_rooms/{4-Buchstaben-Code}/
  stateJson:       JSON.stringify(G)        ← ganzer Game-State
  handsJson/
    seat_0:        JSON.stringify([cards])  ← Hand von Seat 0
    seat_1:        ...
  players/
    {uid}:         { name, seat }
```

### Host-Logik
- **Seat 0 = Host** — dealt Karten, führt KI-Züge aus, schreibt `stateJson`
- Gäste: lesen `stateJson` via `onValue`, schreiben nur ihren eigenen Zug (`playCard`)
- KI-Züge: `aiPlay(hand, trick, rndIdx, seat, numPlayers)` — läuft nur beim Host
- `cppForN(n)`: Karten pro Spieler: 4→9, 5→7, 6→6

### KI-Logik (`aiPlay`)
- Farbzwang erzwingen
- Verlierend → gefährlichste Karte loswerden (Eichel Ober > König > Schilte)
- Gewinnend müssen → niedrigste Karte (ausser letzter Stich → höchste)
- Anspielend → Eichel Ober sofort (Runde 3+5), sonst niedrigste

### Komponenten
```
App               ← Root: screen-State, authUser-State, localStorage-Restore
  LoginScreen     ← Namenseingabe (→ jass_uid + jass_name in localStorage)
  Lobby           ← Raum erstellen / beitreten (4/5/6 Spieler)
  WaitingRoom     ← Warten auf Spieler, Host kann starten / abbrechen
  JassGame        ← Hauptspiel: Spielfeld, Hand, Stich, Score-Panel
```

### localStorage-Keys (jass.html)
| Key | Inhalt |
|-----|--------|
| `jass_uid` | Gast-UID (`g_...`), persistent |
| `jass_name` | Anzeigename, persistent |
| `jass_room` | Aktiver Raumcode (4 Zeichen), temporär |

### AI_NAMES (KI-Spieler)
```js
['Roman', 'Mäggie', 'Cpt. Rotkopf', 'Claude', 'E.B03']
```

---

## Firebase Security Rules

### Aktuell empfohlen (index.html Pfade)
```json
{
  "rules": {
    "jasstafel":  { ".read": "auth != null", ".write": "auth != null" },
    "wins":       { ".read": "auth != null", ".write": "auth != null" },
    "protokoll":  { ".read": "auth != null", ".write": "auth != null" },
    "diskussion": { ".read": "auth != null", ".write": "auth != null" },
    "config": {
      ".read":  "auth != null",
      ".write": "auth.token.email == '[admin1]' || auth.token.email == '[admin2]'"  // E-Mails direkt in index.html
    }
  }
}
```

### jass_rooms — muss offen sein (kein Firebase Auth)
```json
"jass_rooms": {
  ".read": true,
  ".write": true
}
```
Da `jass.html` kein Firebase Auth verwendet, müssen `jass_rooms` ohne Auth-Anforderung les-/schreibbar sein.

---

## Design-Tokens

### index.html / jasstafel.html
```css
--bg:    #0b0b09
--ink:   #f0ece0
--gold:  #f0a830
--line:  rgba(240,236,224,.1)
--muted: rgba(240,236,224,.4)
--mono:  'DM Mono'
--sans:  'Syne'
```

### jass.html
```css
body: gold-braun Hintergrund (linear-gradient)
dark panel: rgba(10,8,3,.86), border rgba(255,255,255,.1)
accent green: #2d6a2d / #4a9e4a / #8be08b  (Lobby/Warteraum)
gold accent: rgba(197,160,33,.x)            (aktive Badges)
text light: #f0ece0
font: 'Plus Jakarta Sans'
```

---

## Git-Workflow

- Branch: `main`
- Git-User: `leasuper`
- Freund hat ebenfalls Push-Zugriff → bei `push rejected` immer `git pull --rebase && git push`
- Commit-Format: `feat:` / `fix:` / `refactor:` + kurze Beschreibung

---

## Bekannte Stolperstellen

1. **Push rejected**: Freund pusht parallel → `git pull --rebase && git push`
2. **"File modified since read"**: Linter/Git ändert Datei zwischen Read und Edit → nochmals lesen vor dem Edit
3. **Firebase Auth in jass.html entfernt**: `firebase-auth-compat.js` ist nicht mehr eingebunden — keine `firebase.auth()` Calls dort verwenden
4. **JSX im Browser**: Kein TypeScript, kein Import/Export — alles globale Variablen
5. **Eichel Ober Sonderregel**: Fangen = +40 Punkte (Runde 3+5), unabhängig wer Stich gewinnt → in `calcScore()` mit `mc.some(c=>c.suit==='Eichel'&&c.rank==='O')` geprüft
