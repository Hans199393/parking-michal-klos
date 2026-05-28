# ARCHITEKTURA STRONY – parkingsobieszewo.pl

> Dokument dla następcy: opisuje co, jak, gdzie i po co w całej aplikacji WWW.  
> Ostatnia aktualizacja: 2026-05-28

---

## 1. Przegląd / TL;DR

Strona to **jednolik HTML** (`index.html`) — zero builda, zero bundlera, zero serwera aplikacyjnego.  
Wszystkie zależności ładowane są z CDN (Alpine.js, Tailwind, Supabase JS).  
Hosting: **GitHub Pages** z domeną własną `parkingsobieszewo.pl`.  
Dane w czasie rzeczywistym płyną ze ścieżki: **Parking.OS → Supabase → Vercel API → polling WWW co 15 s**.

> ⚠️ Plik `TECH_STACK.md` w repozytorium opisuje **planowaną** architekturę (React + Vite + Express), która nigdy nie została wdrożona. Ignoruj go — prawda jest w tym dokumencie.

---

## 2. Stack technologiczny (rzeczywisty)

| Warstwa | Technologia | Uwagi |
|---|---|---|
| Struktura | Vanilla HTML5 | `index.html` ~1600 linii |
| Reaktywność UI | **Alpine.js 3.15.11** (CDN, SRI) | `x-data`, `x-bind`, `x-show`, `x-text` etc. |
| Style | **Tailwind CSS CDN** | Play CDN — nie nadaje się do purge/prod, ale tu nie trzeba |
| HTTP client | natywne `fetch()` | brak axios/jQuery |
| Realtime DB | **Supabase JS 2.104.0** (CDN, SRI) | opcjonalne — tylko lokalnie z `public-config.js` |
| Chat API | Vercel serverless (`parking-messenger-bot.vercel.app`) | |
| Analityka | Google Analytics 4 (`G-QDXGCSBJQ3`) z Consent Mode v2 | |
| Mapy | Google Maps embed (iframe lazy) | |
| Czcionki | Google Fonts — Inter | |

**SRI**: Alpine i Supabase mają `integrity="sha384-..."` + `crossorigin="anonymous"` — nie zmieniaj wersji bez aktualizacji hasha.

---

## 3. Struktura plików repozytorium

```
parking_stronawww/
├── index.html                ← Cała strona publiczna (jedyny plik z logiką)
├── admin/
│   └── index.html            ← TYLKO przekierowanie: "panel przeniesiony do Parking.OS"
├── assets/
│   └── images/               ← Zdjęcia (IMG_7749.jpeg, logo2026.png, galeria)
├── public-config.js          ← GITIGNORED — klucze Supabase (tylko lokalnie, na prod 404)
├── public-config.example.js  ← Szablon do kopiowania
├── .env / .env.example       ← Zmienne środowiskowe (secrets:scan)
├── scripts/
│   └── secret-scan.mjs       ← Skrypt anty-leak (npm run secrets:scan)
├── .githooks/                ← Pre-commit hook (uruchamia secrets:scan)
├── regulamin.html            ← Regulamin parkingu
├── regulamin-en/ua/de.html   ← Wersje językowe regulaminu
├── regulamin-chatbota.html   ← Regulamin chatbota Orzeł
├── polityka-prywatnosci.html ← Polityka prywatności / RODO
├── robots.txt / sitemap.xml  ← SEO
├── CNAME                     ← GitHub Pages custom domain (parkingsobieszewo.pl)
├── package.json              ← Tylko skrypty npm (secrets:scan, hooks:install)
└── parkingos/                ← Archiwalne — ignoruj
```

**Admin panel**: `/admin/index.html` to **martwa strona** z komunikatem. Cała administracja odbywa się w aplikacji desktopowej **Parking.OS** (Tauri/React).

---

## 4. Hosting i deploy

| Element | Wartość |
|---|---|
| Repo GitHub | `https://github.com/Hans199393/parking-michal-klos` |
| Branch produkcyjny | `main` |
| Hosting | GitHub Pages (bezpłatny) |
| Domena własna | `parkingsobieszewo.pl` (plik `CNAME`) |
| Drugi URL | `hans199393.github.io/parking-michal-klos` |

### Jak deployować

```powershell
cd G:\parking_2026\parking_stronawww
# sprawdź sekrety przed pushem (obowiązkowe!)
npm run secrets:scan
git add .
git commit -m "opis zmian"
git push origin main
```

Strona jest live po ~1-2 minutach (GitHub Pages pipeline).  
Nie ma CI/CD — push = deploy.

### Jeśli lokalne main rozjechało się z origin/main

```powershell
git stash                        # zachowaj niezacommitowane zmiany
git reset --hard origin/main     # wymuś sync z origin
git stash pop                    # przywróć zmiany
```

---

## 5. Przepływ danych (runtime)

```
Parking.OS (desktop Tauri)
       │
       │ zapisuje do Supabase `settings` table
       ▼
   Supabase PostgreSQL
       │
       │ API proxy (Vercel serverless)
       ▼
https://parking-messenger-bot.vercel.app/api/public-settings
       │
       │ polling co 15 s (PUBLIC_SETTINGS_POLL_MS)
       ▼
   index.html bootstrap IIFE → dispatchRow() → CustomEvents
       │
       ▼
   Alpine.js app() — aktualizuje UI reaktywnie
```

### Dlaczego przez Vercel a nie bezpośrednio Supabase?

Na produkcji plik `public-config.js` nie istnieje (404 — to celowe, plik jest gitignorowany).  
Bez konfiguracji Supabase strona używa **fallbacku do Vercel API**, które serwuje klucze i dane po stronie serwera bez narażania `SUPABASE_SERVICE_ROLE_KEY` w publicznym JS.

**Ścieżka produkcyjna**: zawsze fallback API (Vercel).  
**Ścieżka lokalna/dev**: `public-config.js` z własnym `supabaseUrl`+`supabaseAnonKey` → bezpośredni realtime Supabase.

---

## 6. Bootstrap state (mechanizm startu)

Problem: Alpine inicjalizuje się po załadowaniu JS. Supabase fetch jest asynchroniczny. Jak uniknąć race condition?

**Rozwiązanie**: globalny bufor `window.__PARKING_BOOTSTRAP_STATE`.

```js
// Wypełniany przez IIFE (bootstrap script w <head>) zanim Alpine wstanie
window.__PARKING_BOOTSTRAP_STATE = {
  settings: {},           // klucze z Supabase: rate_basic, open_from, owner_phone...
  komunikat: null,        // aktywny komunikat (JSON)
  komunikatFingerprint: '',
  extraOpenDays: null,    // daty dodatkowych otwarć
  parkingStatus: null,    // { brakMiejsc: bool }
};

// Wywoływana przez Alpine app.init() po zamontowaniu komponentu
window.__PARKING_REPLAY_BOOTSTRAP_STATE();
// ↑ emituje CustomEvents z buforowanego stanu → Alpine odbiera i wyświetla
```

Dzięki temu nawet jeśli fetch skończy się zanim Alpine wstanie, dane nie giną.

---

## 7. Komunikat banner (system ogłoszeń)

Operator w Parking.OS wpisuje `tytul`, `tresc`, datę `od`/`do` → zapisuje do Supabase `settings.komunikat` (JSON).  
WWW pobiera to przy każdym pollu i emituje event `komunikat-update`.

### Dlaczego banner wracał po zamknięciu (stary bug — NAPRAWIONY)

Poll co 15 s emitował `komunikat-update` → Alpine ustawiał `visible = true` unconditionally → banner wracał mimo kliknięcia ✕.

### Aktualne zachowanie (po fixie commit `626f2d8`)

Dwie warstwy ochrony:

**Warstwa 1 — fingerprint deduplication (bootstrap IIFE):**
```js
function buildKomunikatFingerprint(data) {
  return [data.aktywny?'1':'0', data.tytul, data.tresc, data.od, data.do].join('\u001f');
}
// jeśli fingerprint się nie zmienił → return early, event NIE jest emitowany
if (bootstrapState.komunikatFingerprint === fingerprint) return;
```

**Warstwa 2 — dismiss memory (Alpine component):**
```js
komunikat: {
  visible: false,
  currentFingerprint: '',
  dismissedFingerprint: '',  // ← zapamiętuje co użytkownik zamknął
  ...
}

dismissKomunikat() {
  this.komunikat.dismissedFingerprint = this.komunikat.currentFingerprint;
  this.komunikat.visible = false;
}

// w event listenerze:
this.komunikat.visible = fingerprint !== this.komunikat.dismissedFingerprint;
// banner wraca TYLKO gdy admin zmieni treść/daty (nowy fingerprint)
```

**Cykl życia bannera:**
- Po zamknięciu ✕ → ukryty do przeładowania strony LUB zmiany treści przez admina.
- Zmiana dowolnego pola (tytul/tresc/od/do/aktywny) przez admina = nowy fingerprint = banner znowu widoczny.
- `aktywny = false` → fingerprint zerowany, banner zawsze ukryty.

**Dane komunikatu w Supabase** (`settings` table, key `komunikat`):
```json
{
  "aktywny": true,
  "tytul": "Parking otwarty dodatkowo!",
  "tresc": "W środę 4.06 czynny 9:00–16:00",
  "od": "2026-06-03T06:00:00.000Z",
  "do": "2026-06-04T14:00:00.000Z"
}
```

---

## 8. Alpine.js app() — struktura stanu

Główny komponent (`<html x-data="app()" x-init="init()">`):

```
app() {
  lang              ← aktualny język ('pl'|'en'|'ua'|'de'), persist: localStorage('lang')
  mobileMenu        ← boolean, hamburger menu
  showCookie        ← boolean, RODO banner, persist: localStorage('cookieAccepted')
  brakMiejsc        ← boolean, overlay "BRAK WOLNYCH MIEJSC" (event: parking-status)
  komunikat         ← { visible, tytul, tresc, zakres, currentFingerprint, dismissedFingerprint }
  cennik            ← { rate_basic, rate_reservation, open_from, open_to, owner_phone... }
                       hydrowane z Supabase przez event settings-update
  faq[]             ← lista FAQ z kluczami tłumaczeń
  openingStatus     ← 'open' | 'offseason' | { type:'next', date, days }
  extraOpenDays[]   ← daty dodatkowych otwarć (event: extra-open-days)
  translations      ← obiekt PL/EN/UA/DE z wszystkimi string UI

  t(key)            ← helper tłumaczenia, zwraca translations[lang][key]
  setLang(l)        ← zmienia język, persist localStorage
  computeOpeningStatus()  ← logika kalendarza (pt-nd VI-VIII + extraOpenDays)
  getOpeningLabel()       ← tekst statusu w hero ("Otwarty dziś" / "Sezon: VI-VIII" / "Jutro: ...")
  dismissKomunikat()      ← zamyka banner i zapamiętuje fingerprint
  acceptCookie()          ← accept RODO, grant GA4 consent
  declineCookie()         ← decline, denied GA4 consent
  initMaps()             ← lazy load Google Maps iframe po accept cookie
  init()                 ← nasłuchuje CustomEvents, replay bootstrap state, IntersectionObserver
}
```

Drugi komponent to `chatWidget()` — oddzielny Alpine component dla chat widgetu.

---

## 9. Chat widget (Orzeł AI)

```
chatWidget() {
  open        ← boolean, czy widget otwarty
  messages[]  ← historia czatu, persist: sessionStorage('chat_messages')
  sessionId   ← ID sesji, persist: sessionStorage('chat_session_id')
  API_URL     ← 'https://parking-messenger-bot.vercel.app/api/chat'

  send()              ← POST /api/chat { message, sessionId, lang } → reply
  subscribeRealtime() ← Supabase realtime: INSERT na chat_messages WHERE role='owner'
                         (właściciel może odpisać przez panel bota)
  pollNewMessages()   ← backup polling co 5 s (gdy Supabase realtime niedostępny)
  checkBotStatus()    ← sprawdza bot_paused co 30 s (gdy owner przejął czat ręcznie)
}
```

**Chatbot backend**: projekt `parking_botaimess` (Vercel), używa Groq API (LLM) i Supabase do historii.  
Sesja trzymana w `sessionStorage` — reset po zamknięciu karty.  
Powitanie w 4 językach generowane po stronie frontu (nie API).

---

## 10. Multilingual (PL/EN/UA/DE)

Tłumaczenia trzymane bezpośrednio w `app().translations` w `index.html`.  
Brak zewnętrznych plików i18n.

```js
t('key')  →  this.translations[this.lang]['key']
```

Zmiana języka: `setLang('en')` → `lang` w Alpine + `localStorage.setItem('lang','en')` + `document.documentElement.lang`.  
UI aktualizuje się reaktywnie przez `x-text="t('key')"`.

Strona ma również osobne pliki HTML dla regulaminów: `regulamin-en.html`, `regulamin-ua.html`, `regulamin-de.html`.

---

## 11. Google Analytics 4 + Consent Mode v2

```js
// PRZED załadowaniem skryptu GA:
gtag('consent', 'default', { analytics_storage: 'denied', ad_storage: 'denied' });

// Jeśli user poprzednio zaakceptował cookies:
if (localStorage.getItem('cookieAccepted') === '1') {
  gtag('consent', 'update', { analytics_storage: 'granted' });
}
```

Tracking ID: `G-QDXGCSBJQ3`.  
GA4 NIE zbiera danych przed explicit consent — zgodne z RODO/Consent Mode v2.

---

## 12. Google Maps (lazy load)

Mapa nie ładuje się przy starcie strony — tylko po zaakceptowaniu cookies.  
`initMaps()` w Alpine podmienia `<div id="map-placeholder">` na `<iframe>` z URL Google Maps embed.

---

## 13. Bezpieczeństwo

### Klucze i sekrety

- `public-config.js` → NIGDY nie commitować (`.gitignore`). Zawiera `supabaseAnonKey`.
- Na produkcji ten plik **nie istnieje** — 404 jest oczekiwaną odpowiedzią.
- `SUPABASE_SERVICE_ROLE_KEY` jest tylko po stronie Vercel/bota — nigdy w froncie.
- `npm run secrets:scan` — obowiązkowo przed każdym pushem.
- Git pre-commit hook w `.githooks/` uruchamia scan automatycznie (`npm run hooks:install`).

### Content Security

- Alpine.js i Supabase JS mają SRI (Subresource Integrity) — zmiana CDN/wersji bez aktualizacji hasha złamie stronę.
- Brak inline event handlers poza Alpine (`@click` to Alpine, nie `onclick`).
- Brak `eval()` ani `innerHTML` z zewnętrznych danych.
- Dane z Supabase wyświetlane przez `x-text` (safe, nie `x-html`).

---

## 14. Statusy otwierania — logika kalendarza

```js
computeOpeningStatus() {
  // Regularne dni: pt(5), sb(6), nd(0), miesiące VI-VIII
  // Extra dni z Supabase extra_open_days (aktywowane w Parking.OS)
  // Zwraca: 'open' | 'offseason' | { type: 'next', date, days }
}
```

**Ważne**: godziny `8:00-19:00` są hardcoded w logice kalendarza, ale tekst UI pochodzi z `cennik.open_from`/`cennik.open_to` hydrowanych z Supabase.

---

## 15. CSS — szczegóły

- Custom colors: `navy #1a2d4a`, `teal #4dbfbf`, `sunset #e8622a`, `golden #f5c842`, `cream #f0e8d0`
- CSS variable `--banner-h: 0px` — dynamicznie aktualizowana przez ResizeObserver. Navbar przesuwa się o `var(--banner-h)` gdy komunikat jest widoczny, żeby nie zasłaniać treści.
- `.fade-in` + `IntersectionObserver` — sekcje "wjeżdżają" przy scrollowaniu.
- `#brak-miejsc-banner` — pulsujący overlay na środku ekranu (animation: pulse-banner).

---

## 16. Szybka mapa index.html (linie)

| Zakres | Zawartość |
|---|---|
| 1–72 | `<head>`: meta, structured data, Tailwind, Alpine, Supabase CDN |
| 73–120 | Bootstrap IIFE start: stałe, `bootstrapState`, `buildKomunikatFingerprint` |
| 121–175 | `replayBootstrapState()`, `createSupabaseClient()` |
| 176–270 | `fetchPublicState()`, `dispatchRow()`, `dispatchExtraOpenDays()`, `init()` IIFE |
| 271–300 | GA4 Consent Mode, Google Fonts |
| 301–345 | `<style>` globalne |
| 346–397 | Overlay "Brak miejsc" + **Komunikat banner** + Cookie banner |
| 398–500 | Navbar (desktop + mobile menu) |
| 501–600 | Hero section |
| 601–750 | O nas, Cennik, Godziny, Płatności |
| 751–820 | Dojazd (Google Maps) |
| 821–850 | Galeria |
| 851–875 | Opinie (link do Google) |
| 876–900 | FAQ (accordion) |
| 901–950 | Kontakt, Stopka |
| 951–1050 | Messenger float button, Chat widget HTML |
| 1051–1400 | `<script>`: `function app()` (Alpine component) |
| 1400–1560 | `<script>`: `function chatWidget()` (Alpine chat component) |

---

## 17. Znane pułapki (gotchas)

1. **`public-config.js` 404 na prod** — normalne, nie naprawiaj. Fallback do Vercel API jest celowy.
2. **Tailwind CDN Play** — nie minifikuje, nie purge'uje. Klasy są generowane runtime. Jeśli chcesz prod-grade, potrzebujesz buildu.
3. **SRI hash musi się zgadzać** — zmiana wersji Alpine/Supabase bez aktualizacji `integrity=` złamie stronę (przeglądarka zablokuje skrypt).
4. **Oba fingerprinting** — bootstrap IIFE i Alpine app mają `buildKomunikatFingerprint()`. Logika musi być identyczna w obu miejscach.
5. **Extra open days** — ładowane bezpośrednio z Supabase (nie przez Vercel API), wymaga działającej `public-config.js`. Na produkcji bez tego feature działa, ale extra dni nie odświeżają się w realtimie — tylko przy starcie przez fetchPublicState jeśli Vercel API je zwraca.
6. **Chat session** — `sessionStorage`, reset po zamknięciu karty/okna. User traci historię. To design decision, nie bug.
7. **Admin panel** `/admin/` — jest tylko html-ową karteczką "panel przeniesiony". Nie ma tam żadnej logiki. Nie dotykaj.
8. **Deploy z rozbieżnym historią** — jeśli lokalne main rozjechało się z origin (np. zrobiłeś fix lokalnie i przez worktree), użyj `git reset --hard origin/main` po stashowaniu zmian.

---

## 18. Jak zmieniać treści operacyjne

Wszystkie operacyjne dane (cennik, godziny, status miejsc, komunikaty, extra dni) zmieniasz **wyłącznie w aplikacji Parking.OS** (zakładka Ustawienia → Parking).

Parking.OS zapisuje do Supabase → WWW pobiera przez polling lub realtime → UI odświeża się automatycznie.

**Nigdy** nie edytuj tych wartości bezpośrednio w Supabase — Parking.OS jest source of truth.

---

## 19. Powiązane projekty

| Projekt | Lokalizacja | Rola |
|---|---|---|
| Parking.OS | `g:\parking_2026\parking_os\` | Desktopowa app operator (Tauri + React) |
| Bot Orzeł | `g:\parking_2026\parking_botaimess\` | Vercel API: `/api/chat`, `/api/public-settings` |
| WWW (ten repo) | `g:\parking_2026\parking_stronawww\` | Strona publiczna |
| Konfiguracja | `g:\parking_2026\NIE_PSUĆ_WWW_USTAWIENIA_PUBLICZNE.md` | Zasady runtime dla ustawień |
| Polityka sekretów | `g:\parking_2026\000_NADRZEDNA_POLITYKA_SEKRETOW.md` | Zasady dla kluczy API |
