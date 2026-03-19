# BeautyPro — Kontekst projekta

## Šta je ovo
Single-file HTML interaktivni prototip mobilne aplikacije za upravljanje beauty studiom.
Fajl: `lashpro-prototype.html` (u `/mnt/outputs/` i kao `index.html` na GitHub repou)
GitHub: https://github.com/nenadpoz/lash-demo (token čuvati lokalno, ne commitovati)

## Tehnička osnova
- Vanilla JS + CSS custom properties, ~3600+ linija, bez framework-a
- Phone frame simulacija: 393×852px, `position:absolute` ekrani
- Navigacija: `navStack` + `go(name, idx)` / `goBack()`
- Modali: `.modal-overlay` + `.open` klasa (bottom sheets); full-screen overlay koristi `display:flex/none` direktno
- Tema: `:root.light` CSS override, `setTheme(t)` JS funkcija

## Arhitektura ekrana
- `s-login` — Login/Signup (tab toggle), Google dugme
- `s-profession` — Izbor profesije (9 opcija), pojavljuje se posle login-a
- `s-dashboard` — Kalendar + današnji termini
- `s-clients` — Lista klijenata sa pretragom
- `s-profile` — Profil klijenta (cycle tracker, lash profil, istorija)
- `s-finance` — Finansije (4 tabovi: Prihodi, Troškovi, Rang lista, Cenovnik)
- `s-settings` — Podešavanja (tema, profesija, backup...)

## Navigacija u kodu
```js
const screenIds = {
    login:'s-login', profession:'s-profession', dashboard:'s-dashboard',
    clients:'s-clients', profile:'s-profile', finance:'s-finance', settings:'s-settings'
};
```

## Ključni JS objekti i konstante
```js
const CLIENTS = [...]          // 5 klijenata sa istorijom, cycle datumima, blokadama
const ALL_CLIENTS_PICKER = [...] // 20 klijenata za picker (search dropdown)
const APPOINTMENTS = [...]     // 14 termina (mart 2026, dan=15 je "danas")
const CYCLE_STEPS = [...]      // 4 koraka: Novi set, Kor.1, Kor.2, Kor.3
const SERVICE_PRICES = {       // Cene u dinarima (editabilne kroz modal)
    'new-set':3000, 'c2w':2400, 'c3w':2600, 'c4w':2800,
    'reklamacija':0, 'skidanje':1000, 'lash-lift':2000, 'brow-lift':2000
}
const SVC_LABELS = {...}       // Mapiranje ključeva na nazive usluga
const NADOGRADNJA_KEYS = new Set([...]) // Uzajamno isključive usluge
const PROFESSION_CONFIG = {...} // Feature flags po profesiji
let currentProfession = 'lash' // Trenutno odabrana profesija
let BANK_ACCOUNT = '115-0343043400-23'
```

## Sistem profesija
Posle login-a → `s-profession` ekran → `selectProfession(key)` → `startApp()` → `applyProfessionConfig(key)`

**9 profesija:** lash, nails, depilacija, pedikir, makeup, masaza, obrve, kozmetika, frizura

`PROFESSION_CONFIG[key]` sadrži:
- `showCycleTracker` — boolean (samo lash i obrve imaju)
- `showLashProfile` — boolean (samo lash ima)
- `services` — lista usluga za tu profesiju
- `profileSectionTitle` — naziv sekcije profila

## Karakteristični feature-i po profesiji
- **Lash Artist:** Trenutni ciklus tracker + Lash profil (oblik, dužina, tip, uvoj) — JEDINSTVENO za lash
- **Svi ostali:** Samo zakazivanje, klijenti, finansije, podešavanja (ciklus i lash profil skriveni)

## Servisni picker — mutual exclusion logika
```js
// NADOGRADNJA_KEYS: new-set, c2w, c3w, c4w, reklamacija, skidanje — radio unutar grupe
// lash-lift: isključuje celu NADOGRADNJA grupu
// brow-lift: nezavisan toggle
function toggleSvc(id, key) { ... }
```

## Modali (važno)
- Standard modali: `openModal('m-xxx')` / `closeModal('m-xxx')` → toggle `.open` klasa
- Full-screen QR: `#m-price-qr` — koristi `style.display='flex'/'none'`, NE openModal/closeModal
- Today appt modal: `#m-today-appt` → sadržaj renderuje `renderTodayApptModal(a)`

## Kalendar i termini
- `buildCalendar()` — crta cal grid, klikabilni dani
- `dayClick(day)` — otvara `m-day-detail`:
  - Prošlost (day < 15): klik → `go('profile', clientIdx)`
  - Danas/budućnost (day ≥ 15): klik → `openTodayAppt(a.id)`
- Termini na danas (day=15): 4 termina (1 otkazan, 2 aktivna, 1 završen)

## Today Appointment modal (m-today-appt)
3 stanja: cancelled (show detalji + "Zakaži novi"), completed (read-only), active (editable)
- "Preuredi" dugme → inline date+time edit
- "Otkaži termin" → `cancelTodayAppt()` → otvara `m-cancel-appt`
- "Završi sesiju" → `completeTodayAppt()` → čuva cenu i usluge

## Finansije
- 4 tabovi: Prihodi | Troškovi | Rang lista | Cenovnik
- Cenovnik: editabilne cene, QR po usluzi (full-screen), račun banke sa edit opcijom
- Rang lista: `buildTopClients()` — sortira CLIENTS po total zaradi, dodaje emoji tier

## Cycle tracker (lash-specific)
- 4 koraka: Novi set → Kor.1 → Kor.2 → Kor.3
- Klikom na step → tooltip sa datumom i intervalima (`toggleCycleTip(idx)`)
- Pokazuje "Xd. ranije" u naslovu sekcije
- `done-all` cycleStep → prikazuje ⚠️ banner "Potreban novi set"

## Ciklus cena (din)
- Novi set: 3.000 din
- Korekcija 2 ned: 2.400 din
- Korekcija 3 ned: 2.600 din
- Korekcija 4 ned: 2.800 din
- Reklamacija: 0 (besplatno)
- Skidanje: 1.000 din
- Lash lift: 2.000 din
- Brow lift: 2.000 din

## Stil i tema
- Default: tamna tema (`--bg:#0D0D0D`, `--gold:#C8A84B`)
- Light tema: `:root.light` override
- Valuta: din (RSD)
- Jezik: srpski

## Poznati edge case-ovi
- `replace_all` na unique stringu može slučajno pogoditi više mesta — uvek biti pažljiv
- Full-screen modal (`#m-price-qr`) ne koristi `.open` klasu — posebno tretirati
- `cancelTodayAppt()` reuse-uje postojeći `openCancelAppt(id)` flow
- CLIENTS[idx].cycleDates prati stvarne datume za tooltip

## Git workflow
```bash
cp /sessions/jolly-trusting-ride/mnt/outputs/lashpro-prototype.html lash-demo/index.html
cd lash-demo && git add index.html && git commit -m "..." && git push
```
