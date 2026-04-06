# Parking Michał Kłos – Instrukcja konfiguracji

## Co jest gotowe
- Pełna strona jednostronicowa (PL/EN/UA/DE)
- Panel admina (`/admin`) – włączanie/wyłączanie banera "Brak miejsc"
- Wszystkie sekcje: Hero, O nas, Cennik, Dojazd, Galeria, Opinie, FAQ, Kontakt, Stopka
- Floating Messenger chat button
- Cookie/RODO banner
- SEO (meta tagi, structured data)
- Mobile-first, animacje fade-in on scroll

---

## KROKI DO UZUPEŁNIENIA

### 1. Firebase Firestore (wymagany dla banera "Brak miejsc")

1. Wejdź na https://console.firebase.google.com
2. Kliknij "Add project" → nazwa np. `parking-michal-klos`
3. Po utworzeniu: `Build` → `Firestore Database` → Create database → Start in **test mode**
4. W Firestore utwórz kolekcję `parking` → dokument `status` → pole `brakMiejsc` (boolean) → wartość `false`
5. Wróć do głównej konsoli → kliknij ikonę `</>` (Web app) → nadaj nazwę → zarejestruj
6. Skopiuj `firebaseConfig` i wklej w **obu** plikach:
   - `index.html` (linia z `const firebaseConfig = {`)
   - `admin/index.html` (ta sama linia)

Zamień placeholders:
```
FIREBASE_API_KEY
FIREBASE_AUTH_DOMAIN
FIREBASE_PROJECT_ID
FIREBASE_STORAGE_BUCKET
FIREBASE_MESSAGING_SENDER_ID
FIREBASE_APP_ID
```

### 2. Google Analytics GA4

1. Wejdź na https://analytics.google.com
2. Utwórz konto → Właściwość → Strumień danych Web
3. Skopiuj ID pomiaru (format: `G-XXXXXXXXXX`)
4. W pliku `index.html` zamień **dwa** wystąpienia `GA_MEASUREMENT_ID` na swoje ID

### 3. Mapa Google Maps (embed)

Aktualna mapa jest placeholderem. Aby ustawić prawidłową mapę:
1. Wejdź na https://maps.google.com
2. Znajdź swój parking (ul. Turystyczna 69, Wyspa Sobieszewska)
3. Kliknij `Udostępnij` → `Umieść mapę` → skopiuj kod `<iframe>`
4. W `index.html` zamień istniejący `<iframe>` w sekcji `#dojazd` na skopiowany kod

### 4. Facebook Messenger Customer Chat (opcjonalnie – pełny widget)

Aktualnie jest pływający przycisk kierujący do m.me/...
Jeśli chcesz pełny widget czatu Messengera na stronie:
1. Wejdź na https://www.facebook.com/business/help
2. Szukaj "Customer Chat Plugin"
3. Wygeneruj kod dla swojej strony FB i dodaj przed `</body>` w `index.html`

### 5. Hosting (home.pl / nazwa.pl)

1. Kup domenę na home.pl lub nazwa.pl
2. Włącz hosting (plan Basic wystarczy)
3. Wgraj przez FTP lub panel pliki:
   - `index.html`
   - `assets/` (folder z obrazkami)
   - `admin/` (folder z panelem admina)
4. Strona będzie dostępna pod Twoją domeną
5. Panel admina: `twojadomena.pl/admin/`

---

## Lokalne podglądanie strony

Otwórz plik `index.html` w przeglądarce:
- Kliknij prawym na plik → "Otwórz za pomocą" → Chrome/Firefox
- LUB: w VS Code zainstaluj rozszerzenie "Live Server" → kliknij prawym na index.html → "Open with Live Server"

**Panel admina lokalnie:** Otwórz `admin/index.html` w przeglądarce.

---

## Struktura projektu

```
parking-michal-klos/
├── index.html          ← strona główna
├── admin/
│   └── index.html      ← panel admina (hasło: Poczesi1!)
├── assets/
│   └── images/
│       ├── logo2026.png
│       ├── IMG_7749.jpeg  ← tło hero
│       ├── IMG_7747.jpeg
│       ├── IMG_7748.jpeg
│       ├── IMG_7749 (1).jpeg
│       ├── IMG_7753.jpeg
│       └── 20250429_192814.jpg
└── README.md
```
