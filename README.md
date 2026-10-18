# IzboriApp — Hibridni Sistem za Projekciju Izbornih Rezultata

> **Statističko modelovanje, predviđanje prelaska cenzusa (3.00%) i raspodela 250 poslaničkih mandata po D'Hondtovom sistemu u realnom vremenu za parlamentarne izbore u Republici Srbiji.**

---

## 📌 Pregled sistema

**IzboriApp** je analitička i operativna platforma razvijena za praćenje izborne noći. Sistem kombinuje istorijsku bazu svih **8.273 biračka mesta** u Srbiji (192 opštine, 9 makro-stratuma) sa živim unosom zapisnika sa terena i zvaničnih RIK feed-ova.

Platforma rešava dva ključna izazova:
1. **Predviđanje rezultata tradicionalnih i koalicionih lista** putem višedimenzionalnih latentnih koordinata (5D TruncatedSVD) i Neymanovog stratifikovanog ponderisanja.
2. **Projekcija novih lista bez istorijskog učešća ("Zero-Baseline" liste)** u ranoj fazi izborne noći bez prekoračenja margine greške.

---

## 🚀 Ključne funkcionalnosti

- **Izbor optimalnog reprezentativnog uzorka ($K^* = 200$ biračkih mesta)**:
  - Algoritam koristi *Hat-Matrix* statističku polugu (leverage) i ponderisanu ocenu brzine izveštavanja (Pareto frontier solver) za automatski odabir najinformativnijih 200 biračkih mesta u 9 regiona.
  - Obezbeđuje grešku procene manju od **0.35%** pre pristizanja većine biračkih mesta.
- **Četvorostepeni režim projekcije u realnom vremenu**:
  - `Režim 0 (A Priori)`: Početno stanje pre pristizanja zapisnika.
  - `Režim 1 (Hladni start, 1–29 BM)`: Ponderisani regionalni prosek sa širokim intervalima poverenja.
  - `Režim 2 (SVD Hibridni model, 30–85% BM)`: Ridge regresija u 5D latentnom prostoru biračkih mesta uz Neymanovu stratifikaciju.
  - `Režim 3 (Asimptotsko zaključavanje, >85% BM)`: Konvergencija ka konačnim zvaničnim rezultatima.
- **D'Hondtova raspodela 250 mandata i post-izborne koalicije**:
  - Automatska primena prirodnog cenzusa za manjinske liste i cenzusa od 3.00% za standardne liste.
  - Analiza graničnih mandata (trka oko 250. mesta) i simulator parlamentarne većine ($\ge 126$ poslanika) sa Banzhafovim indeksom koalicione snage.
- **Automatizovana kontrola i karantin zapisnika**:
  - Dvosmerno prepoznavanje ćiriličnih i latiničnih naziva opština i biračkih mesta.
  - SHA-256 deduplikacija pristiglih paketa.
  - Detekcija i izolacija matematički nevalidnih zapisnika (veći broj glasova od broja upisanih birača, zbir glasova ne odgovara broju važećih listića).
- **Moderan i nenametljiv korisnički interfejs**:
  - Minimalistički tamni dizajn optimizovan za rad u studiju i operativnom centru tokom izborne noći.
  - Mogućnost simulacije priliva glasova po talasima (5%, 15%, 50%, 100%) i direktnog unosa CSV/JSON/HTML zapisnika.

---

## 🛠️ Tehnološki stek

- **Frontend & Server**: [Next.js](https://nextjs.org/) (App Router), [React](https://react.dev/), TypeScript
- **Baza podataka**: SQLite u Write-Ahead Logging (WAL) režimu (`node:sqlite` ugrađeni drajver)
- **Matematičko i statističko jezgro**: Truncated SVD, Neyman Optimal Allocation, Ridge Regression, D'Hondt algoritam
- **Stilovi**: Tailwind CSS / Vanilla Modern Dark Slate

---

## 📦 Instalacija i pokretanje

### Preduslovi
- [Node.js](https://nodejs.org/) (verzija 22 ili novija, preporučeno zbog ugrađene SQLite podrške)
- npm ili yarn / pnpm

### 1. Kloniranje repozitorijuma
```bash
git clone https://github.com/VASE_KORISNICKO_IME/IzboriApp.git
cd IzboriApp
```

### 2. Instalacija zavisnosti
```bash
npm install
```

### 3. Pokretanje u razvojnom režimu (Development)
```bash
npm run dev
```
Aplikacija će biti dostupna na adresi: `http://localhost:3000`

### 4. Pravljenje produkcione verzije (Production Build)
```bash
npm run build
npm run start
```

---

## 🧪 Pokretanje testova

Projekat poseduje sveobuhvatnu test bazu koja pokriva integritet baze podataka, rad statističkog modela i validaciju unosa:

```bash
# Pokretanje svih testova
npm test

# Pokretanje pojedinačnih celina:
npm run test:schema2026      # Verifikacija 2026 SQLite šeme i transakcija
npm run test:sentinel        # Test optimizacije reprezentativnog uzorka (200 BM)
npm run test:projection2026  # Test matematičkog modela projekcije i D'Hondta
npm run test:ingest2026      # Test parsiranja zapisnika i karantinske validacije
npm run test:sentinel2026    # Integracioni test operativnih API ruta
```

---

## 📂 Struktura projekta

```text
IzboriApp/
├── data/                       # Istorijska master baza biračkih mesta (SQLite)
├── scripts/                    # Test skripte i simulacije za izbornu noć
├── src/
│   ├── app/
│   │   ├── api/
│   │   │   ├── ingest/2026/    # API za prijem i validaciju zapisnika sa terena
│   │   │   └── operations/     # API za kontrolu reprezentativnog uzorka i simulaciju
│   │   ├── globals.css         # Globalni stilovi i dizajn tema
│   │   ├── layout.tsx          # Koreni layout aplikacije
│   │   └── page.tsx            # Glavna kontrolna tabla (Dashboard)
│   ├── components/
│   │   └── panels/             # Korisnički paneli (Rezultati, Mandati, Koalicije, Uzorak, Opštine)
│   ├── lib/
│   │   ├── db/                 # SQLite WAL singleton, šeme i repozitorijumi
│   │   ├── engine/             # Statistički model, SVD hibrid i D'Hondt
│   │   └── ingest/             # Resolver biračkih mesta i karantinski validator
│   └── store/                  # Stanje aplikacije na klijentu
├── .gitignore                  # Git ignorisani fajlovi
├── package.json
└── tsconfig.json
```

---

## 📄 Licenca

Ovaj projekat je licenciran pod [MIT licencom](LICENSE).
