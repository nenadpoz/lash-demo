# BeautyPro — Plan razvoja

> Dokument kreiran: Mart 2026
> Status: U planiranju — razvoj još nije počeo
> Demo prototip: `index.html` (single-file HTML, vanilla JS)

---

## Vizija proizvoda

BeautyPro je aplikacija za upravljanje beauty studiom namenjena individualnim artistima i salonima u Srbiji i ex-Yu regionu, sa planiranim proširenjem na šire tržište. Cilj je da bude lokalizovana alternativa globalnim rešenjima (Fresha, Vagaro, GlossGenius) — srpski jezik, dinari, domaći UX, po pristupačnoj ceni.

---

## Ključne odluke

| Odluka | Izbor | Razlog |
|--------|-------|--------|
| Redosled razvoja | Mobile prvo | Prototip je mobile-first |
| Platforme | iOS + Android | Expo podržava oba iz jednog codebase-a |
| Multi-tenant | Single-user prvo | Lakši start, arhitektura podržava upgrade |
| Monetizacija | 30 dana free trial → pretplata | Niska barijera za probanje |
| Region | Srbija/ex-Yu prvo | Poznato tržište, srpski default |
| i18n | Od prvog dana | Sprečava skupo refaktorisanje kasnije |
| Expo workflow | Managed | Lakši DX, dovoljan za naš use case |
| Shop/Marketplace | Posle v1 deploya | Kompleksna funkcionalnost, ne blokira core |

---

## Tech Stack

### Mobile (primarna platforma)
- **Expo (Managed Workflow)** — React Native za iOS + Android
- **TypeScript** — type safety od prvog dana
- **Expo Router** — file-based navigacija (kao Next.js za mobile)

### Web (faza 8)
- **Next.js 14+** — SSR, App Router, Vercel deploy
- **React.js** — UI komponente (shared sa `packages/ui/` gde je moguće)
- **TypeScript**

### Monorepo
- **Turborepo** — orchestracija workspaceova, shared build cache

### Backend & Baza
- **Supabase** — PostgreSQL + Auth + Storage + Realtime
  - Row Level Security (RLS) — svaki artist vidi samo svoje podatke
  - Google OAuth out-of-the-box
  - Generisani TypeScript tipovi iz šeme

### State Management
- **Zustand** — globalni state (auth, profession, settings)
- **TanStack Query (React Query)** — server state, cache, sync

### Notifikacije & Komunikacija
- **Expo Push Notifications** — in-app notifikacije
- **Infobip** — SMS podsetnici (podržava ex-Yu region)
- **Resend** — transakcioni email (potvrde, fakture)

### Plaćanje & Pretplata
- **RevenueCat** — unified billing za iOS App Store + Google Play
  - Jedan SDK za obe platforme
  - Webhook sync sa Supabase

### i18n
- **i18next + react-i18next** — prevodi
- **Intl.NumberFormat** — formatiranje valuta i datuma per-locale

### Deployment
- **EAS Build** (Expo Application Services) — build + submit na store-ove
- **Vercel** — web app (faza 8)

### Shop/Marketplace (faza 9+)
- **Stripe Connect** — isplate prodavcima, provizija za BeautyPro (alternativa: Payspot/NestPay za ex-Yu)
- **Cloudinary / Supabase Storage** — upload slika proizvoda
- **Algolia / pg_trgm** — pretraga proizvoda po tekstu

---

## Arhitektura monorepa

```
beautypro/
├── apps/
│   ├── mobile/                  → Expo (React Native) app
│   │   ├── app/                 → Expo Router ekrani
│   │   │   ├── (auth)/          → login, role-select, signup, profession
│   │   │   ├── (app)/           → dashboard, clients, finance, settings
│   │   │   └── (shop)/          → shop, product, seller-profile  ← faza 9
│   │   ├── components/          → mobile-specific komponente
│   │   └── app.json
│   │
│   └── web/                     → Next.js app (faza 8)
│       ├── app/                 → Next.js App Router
│       └── components/          → web-specific komponente
│
└── packages/
    ├── shared/                  → deljena poslovna logika
    │   ├── hooks/               → useClients, useAppointments, useFinance...
    │   │                           useProducts, useOrders, useReviews  ← faza 9
    │   ├── store/               → Zustand store
    │   ├── utils/               → formatPrice, formatDate, calcCycleStep...
    │   └── constants/           → PROFESSION_CONFIG, SERVICE_KEYS...
    │                               ROLE_CONFIG  ← faza 9
    │
    ├── ui/                      → deljene UI komponente
    │   ├── Button/
    │   ├── Card/
    │   ├── Input/
    │   ├── Modal/
    │   ├── StarRating/           ← faza 9
    │   ├── ReviewCard/           ← faza 9
    │   └── ...
    │
    ├── supabase/                → Supabase klijent + generisani tipovi
    │   ├── client.ts
    │   ├── types.ts             → auto-generisano iz šeme
    │   └── queries/             → reusable query funkcije
    │                               + productQueries, orderQueries  ← faza 9
    │
    └── i18n/                    → internacionalizacija
        ├── index.ts
        └── locales/
            ├── sr.json          → srpski (default)
            ├── en.json          → engleski
            ├── hr.json          → hrvatski
            └── bs.json          → bosanski
```

**Princip deljenja koda:** Web i mobile dele ~65-70% koda kroz `packages/`. Samo UI sloj i navigacija su odvojeni jer se vizuelno i UX-no razlikuju.

---

## Supabase — šema baze podataka

Dizajnirana da podržava **single-user danas**, sa lakim upgradeom na **multi-tenant (salon sa više artista) u budućnosti** — bez potrebe za migracijom podataka.

```sql
-- ── Profil korisnika (1:1 sa auth.users) ─────────────────────────
profiles
  id              uuid PK
  user_id         uuid FK → auth.users
  full_name       text
  email           text
  avatar_url      text
  salon_name      text
  role            text        -- 'artist' | 'seller' | 'both'   ← faza 9
  professions     text[]      -- ['lash','nails',...] multi-prof  ← v1 update
  active_profession text      -- trenutno aktivna profesija
  plan            text        -- 'trial' | 'solo' | 'studio'
  trial_ends_at   timestamptz
  bank_account    text
  currency        text        -- 'RSD' | 'EUR' | 'HRK' | ...
  locale          text        -- 'sr' | 'en' | 'hr' | 'bs'
  created_at      timestamptz

-- ── Klijenti ─────────────────────────────────────────────────────
clients
  id              uuid PK
  profile_id      uuid FK → profiles   ← owner
  full_name       text
  phone           text
  email           text
  instagram       text
  notes           text
  blocked         boolean
  block_reason    text
  cancellations   int
  total_earned    numeric
  -- Lash-specific (null za ostale profesije)
  lash_shape      text
  lash_length     text
  lash_type       text
  lash_curl       text
  -- Cycle
  cycle_step      text        -- 'new-set' | 'c1' | 'c2' | 'c3' | 'done-all'
  created_at      timestamptz

-- ── Termini ──────────────────────────────────────────────────────
appointments
  id              uuid PK
  profile_id      uuid FK → profiles
  client_id       uuid FK → clients
  date            date
  time            time
  status          text        -- 'scheduled' | 'completed' | 'cancelled'
  services        text[]      -- ['new-set', 'lash-lift']
  amount          numeric
  notes           text
  cancelled_by    text
  cancel_reason   text
  created_at      timestamptz

-- ── Ciklus datumi (lash-specific) ────────────────────────────────
cycle_dates
  id              uuid PK
  client_id       uuid FK → clients
  step            text
  date            date
  days_from_prev  int

-- ── Troškovi ─────────────────────────────────────────────────────
expenses
  id              uuid PK
  profile_id      uuid FK → profiles
  category        text
  description     text
  amount          numeric
  date            date
  created_at      timestamptz

-- ── Cene usluga (per artist, editabilne) ─────────────────────────
service_prices
  id              uuid PK
  profile_id      uuid FK → profiles
  service_key     text        -- 'new-set' | 'c2w' | ...
  label           text        -- prilagođeni naziv (artist može da preimenuje)
  price           numeric

-- ── Pretplata (sync sa RevenueCat webhook) ───────────────────────
subscriptions
  id              uuid PK
  profile_id      uuid FK → profiles
  status          text        -- 'trialing' | 'active' | 'expired' | 'cancelled'
  plan            text        -- 'solo' | 'studio'
  current_period_start  timestamptz
  current_period_end    timestamptz
  provider        text        -- 'apple' | 'google'
  provider_id     text

-- ════ SHOP tablice (faza 9) ══════════════════════════════════════

-- ── Prodavac profil (1:1 sa profiles gde role='seller'|'both') ───
seller_profiles                                          -- faza 9
  id              uuid PK
  profile_id      uuid FK → profiles
  shop_name       text
  shop_description text
  logo_url        text
  verified        boolean     -- BeautyPro može verifikovati prodavca
  commission_pct  numeric     -- % provizije za BeautyPro (default 10%)
  stripe_account_id text      -- Stripe Connect account (isplate)
  avg_rating      numeric     -- kalkulisano, refresh trigger-om
  review_count    int
  created_at      timestamptz

-- ── Proizvodi ─────────────────────────────────────────────────────
products                                                 -- faza 9
  id              uuid PK
  seller_id       uuid FK → seller_profiles
  name            text
  brand           text
  description     text
  category        text        -- 'lash' | 'nails' | 'nega' | 'alati' | 'kozmetika'
  price           numeric
  original_price  numeric     -- NULL ako nema popusta
  stock_count     int
  images          text[]      -- array URL-ova
  is_active       boolean
  avg_rating      numeric     -- kalkulisano
  review_count    int
  created_at      timestamptz

-- ── Recenzije (proizvoda i prodavca) ─────────────────────────────
reviews                                                  -- faza 9
  id              uuid PK
  profile_id      uuid FK → profiles   -- autor recenzije
  product_id      uuid FK → products   -- NULL ako je recenzija prodavca
  seller_id       uuid FK → seller_profiles  -- NULL ako je recenzija proizvoda
  rating          int         -- 1-5
  text            text
  seller_reply    text        -- prodavac može odgovoriti, NE brisati
  seller_replied_at timestamptz
  created_at      timestamptz
  -- Constraint: ne može biti obrisana od strane prodavca (RLS policy)

-- ── Narudžbine ───────────────────────────────────────────────────
orders                                                   -- faza 9
  id              uuid PK
  buyer_id        uuid FK → profiles
  seller_id       uuid FK → seller_profiles
  status          text        -- 'pending' | 'confirmed' | 'shipped' | 'delivered' | 'cancelled'
  total_amount    numeric
  commission_amount numeric   -- BeautyPro provizija
  seller_amount   numeric     -- iznos za prodavca (total - komisija)
  delivery_address text
  notes           text
  created_at      timestamptz

-- ── Stavke narudžbine ────────────────────────────────────────────
order_items                                              -- faza 9
  id              uuid PK
  order_id        uuid FK → orders
  product_id      uuid FK → products
  quantity        int
  unit_price      numeric
  subtotal        numeric
```

### Multi-tenant upgrade (kad dođe vreme)
Dodaje se `salons` tabela, `profile_id` dobija `role` kolonu (`owner` / `staff`), RLS policy se ažurira. **Podaci se ne diraju, migracija je aditivna.**

### Shop skaliranje (faza 9+)
Arhitektura je dizajnirana da Shop tablice (`seller_profiles`, `products`, `reviews`, `orders`) budu **aditivne** — ne menjaju postojeće artist tablice. RLS policy za recenzije osigurava da prodavac ne može brisati recenzije (`USING (auth.uid() = buyer_profile_id)`). Provizija se može menjati per-seller u `seller_profiles.commission_pct`.

---

## Sistem profesija i uloga

### Profesije
Profession config se čuva u `packages/shared/constants/professions.ts` i određuje koje feature-e vidi određeni artist.

```typescript
type ProfessionConfig = {
  key: string
  name: string               // lokalizovano kroz i18n
  icon: string
  showCycleTracker: boolean  // samo lash i obrve
  showLashProfile: boolean   // samo lash
  profileSectionTitle: string
  serviceKeys: string[]      // usluge dostupne ovoj profesiji
}
```

**9 profesija u v1:**
`lash` | `nails` | `depilacija` | `pedikir` | `makeup` | `masaza` | `obrve` | `kozmetika` | `frizura`

Korisnik može imati **više aktivnih profesija** (multi-profession). Između njih se prebacuje u dashboardu — aplikacija prilagođava UI aktivnoj profesiji.

### Uloge (faza 9)
```typescript
type UserRole = 'artist' | 'seller' | 'both'
```

| Uloga | Pristup | Navigacija |
|-------|---------|-----------|
| `artist` | Kalendar, Klijenti, Finansije, Settings | Standardna 4/5-tab navigacija |
| `seller` | Prodavnica, Istraži, Statistika, Settings | Seller navigacija (bez Kalendara) |
| `both` | Sve od gornjeg + Shop tab u artist navu | Unified — switch kontekst |

- Uloga se bira pri registraciji i može se proširiti iz Settings (artist → both, besplatno)
- `seller`-only nalog **nema** pristup Calendar/Client/Finance modulima

---

## Monetizacija

### Artist (pretplata)
```
Free Trial     → 30 dana, sve funkcije dostupne
Solo Plan      → ~500 din/mes  ili  ~4.500 din/god (2 meseca gratis)
               → 1 artist, sve funkcije
Studio Plan    → faza budućeg razvoja, više artista u salonu
```

### Prodavac/Shop (provizija, faza 9)
```
Registracija prodavca   → Besplatno
Listanje proizvoda      → Besplatno, bez limita
BeautyPro provizija     → 8–12% po transakciji (tačan % TBD)
Isplate prodavcima      → Stripe Connect (direktna isplata)
```
Prodavac koji je i Artist ne plaća ekstra za Shop — samo Artist pretplatu.

### Implementacija
- **RevenueCat** upravlja billing-om na iOS i Android (artist pretplata)
- Na kraju triala → Paywall ekran (ne pre)
- Ako ne plati → **Read-only mode** (vidi istoriju, ne može dodavati)
- RevenueCat webhook → Supabase `subscriptions` tabela (real-time sync)
- **Stripe Connect** (faza 9) → isplate prodavcima, automatic provizija split

---

## i18n strategija

- Svi stringovi u kodu kroz `t('namespace.key')` — **bez hardcodovanih srpskih stringova**
- `sr.json` je default i razvija se paralelno sa kodom
- Valuta: čuva se `currency` u `profiles`, formatira se kroz `Intl.NumberFormat(locale, { currency })`
- Datumi: `Intl.DateTimeFormat(locale)` — automatski prilagođeni formati
- Dodavanje novog jezika = novi JSON fajl, bez code promene

---

## Faze razvoja

### Faza 1 — Infrastruktura
- [ ] Turborepo monorepo scaffold
- [ ] Supabase projekt, šema baze, RLS politike
- [ ] Expo app (Managed), Expo Router setup
- [ ] TypeScript, ESLint, Prettier konfiguracija
- [ ] `packages/supabase` — klijent + generisani tipovi
- [ ] `packages/i18n` — srpski default, i18next setup
- [ ] Dark/light tema setup (design tokens)

### Faza 2 — Autentikacija
- [ ] Login ekran (email/lozinka)
- [ ] Signup ekran (email/lozinka/ponovi)
- [ ] Google OAuth (Supabase)
- [ ] Ekran za izbor **uloge** (Artist / Prodavac / Oboje)
- [ ] Ekran za izbor **profesija** (multi-select, samo za Artist/Both)
- [ ] Protected route logika (redirect ako nije ulogovan)
- [ ] Trial period start
- [ ] `profiles.role` i `profiles.professions[]` upisani pri onboardingu

### Faza 3 — Klijenti
- [ ] Lista klijenata (search, filter blokiran/aktivan)
- [ ] Dodaj novog klijenta
- [ ] Profil klijenta
- [ ] Izmeni klijenta (info + lash profil)
- [ ] Blokiranje/deblokiranje
- [ ] Istorija termina po klijentu

### Faza 4 — Kalendar i termini
- [ ] Kalendarski prikaz (mesec)
- [ ] Zakazivanje termina (Book modal)
- [ ] Today lista sa statusima
- [ ] Detalji termina (otkaži, završi, preuredi)
- [ ] Cycle tracker (lash-specific, conditional rendering)
- [ ] Overlap detekcija

### Faza 5 — Finansije
- [ ] Prihodi sa filterom dan/nedelja/mesec
- [ ] Troškovi (CRUD)
- [ ] Grafikon prihoda (SVG bar chart, time range opcije)
- [ ] Cenovnik (editabilne cene per artist)
- [ ] QR za plaćanje (račun banke)
- [ ] Rang lista klijenata

### Faza 6 — Notifikacije, pretplata, podešavanja
- [ ] Push notifikacije (Expo)
- [ ] SMS podsetnici 24h pre termina (Infobip)
- [ ] RevenueCat integracija + Paywall ekran
- [ ] Read-only mode po isteku triala
- [ ] Settings (tema, jezik, valuta, profil salona, backup)
- [ ] Cloud backup (Supabase Realtime)

### Faza 7 — Store release (v1.0)
- [ ] EAS Build setup
- [ ] App Store (iOS) — screenshots, opis, review
- [ ] Google Play (Android) — screenshots, opis, review
- [ ] Post-launch monitoring (crash reporting, analytics)
- [ ] **v1.0 live** — artist features kompletne, bez Shop-a

### Faza 8 — Web aplikacija
- [ ] Next.js scaffold, isti `packages/`
- [ ] Supabase SSR autentikacija
- [ ] Svi ekrani adaptirani za desktop layout
- [ ] Deploy na Vercel

### Faza 9 — Shop & Marketplace
> Počinje tek nakon stabilnog v1.0 deploya na store-ovima.

**9a — Seller onboarding i proizvodi**
- [ ] Ekran za odabir Prodavac uloge (pri registraciji ili iz Settings)
- [ ] Seller profil setup (naziv, opis, logo)
- [ ] CRUD za proizvode (naziv, opis, kategorija, cena, slike)
- [ ] Supabase tablice: `seller_profiles`, `products`
- [ ] Seller dashboard ekran (statistike, lista proizvoda)
- [ ] Stripe Connect integracija (onboarding prodavca)

**9b — Shop browsing (artist side)**
- [ ] Shop tab u artist navigaciji (samo za `both` ulogu)
- [ ] Browse ekran sa kategorijama i pretragom
- [ ] Detalj proizvoda (opis, slike, rating, recenzije)
- [ ] Seller profil (javni, sa svim proizvodima i ukupnim ratingom)

**9c — Recenzije i rating sistem**
- [ ] Prikaz recenzija na proizvodu i profilu prodavca
- [ ] Forma za pisanje recenzije (1–5 ⭐ + tekst)
- [ ] Prodavac može **odgovoriti** na recenziju, ali je **ne može obrisati**
- [ ] RLS policy: `DELETE` zabranjen za prodavca na tuđim recenzijama
- [ ] Automatski recalc `avg_rating` triggerom na `reviews` tabeli

**9d — Narudžbine i plaćanje**
- [ ] Narudžbina flow (odabir, pregled, potvrda)
- [ ] Supabase tablice: `orders`, `order_items`
- [ ] Stripe provizija split (BeautyPro %, prodavac %)
- [ ] Notifikacija prodavcu (nova narudžbina)
- [ ] Notifikacija kupcu (potvrda, isporuka)
- [ ] Order history (buyer + seller view)

**9e — Store release v2.0**
- [ ] Ažuriranje iOS/Android listinga (nove screenshots, opis Shop feature-a)
- [ ] Push notifikacije za Shop events (nova recenzija, nova narudžbina)

---

## Konkurencija (referenca)

| Aplikacija | Tržište | Cena | Naša prednost |
|-----------|---------|------|---------------|
| Fresha | Globalno | Besplatno + % | Srpski jezik, RSD, lokalni UX |
| Vagaro | SAD/CA | ~$30/mes | Jeftiniji, lokalizovan |
| GlossGenius | SAD | ~$24/mes | Lokalizovan, ex-Yu fokus |
| Square Appointments | SAD | Besplatno/paid | Lokalizovan |
| Zenoti | Enterprise | Skupo | Pristupačan za solo artiste |

---

## Napomene

- Demo prototip (`index.html`) ostaje kao referenca za UX/dizajn tokom razvoja — uključuje Shop demo
- Svaka faza završava sa funkcionalnim build-om koji se može testirati
- Git flow: `main` (production), `develop` (aktivni razvoj), feature grane
- Repo: https://github.com/nenadpoz/lash-demo (demo) → novi repo za pravu app
- **Prioritet je v1.0 (faze 1–7)** — Shop se ne razvija pre nego što je core artist app stabilan na store-ovima
- Faza 9 (Shop) može krenuti paralelno sa Fazom 8 (Web) ako ima kapaciteta
