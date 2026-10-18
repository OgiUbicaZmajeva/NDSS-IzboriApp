# IzboriApp — Status implementacije i verifikacije po fazama

Ovaj dokument prati sve izgrađene module, arhitektonske izmjene i rezultate funkcionalnih testova nakon svakog koraka.

---

## Faza 1: Temelj (Tipovi podataka i In-Memory SQLite Singleton) — ZAVRŠENO

### Kreirani i ažurirani fajlovi:
- [`data/election_master_baseline.db`](file:///c:/Users/ognje/Desktop/IzboriApp/data/election_master_baseline.db) — Kanonski SQLite artefakt (`1.937.408 B`, `8.273` biračka mjesta, `31` kolona).
- [`src/types/baseline.types.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/types/baseline.types.ts) — Strogi tipski ugovori za 31 kolonu SQLite tabele, 9 makro-strata (`BG-URB`, `BG-SUB`, `VOJ-URB`, `VOJ-RUR`, `SUM-URB`, `SUM-RUR`, `JUG-URB`, `JUG-RUR`, `SPEC-EXT`), 5 SVD latentnih dimenzija i Neymanove alokacione težine.
- [`src/types/ingest.types.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/types/ingest.types.ts) — Tipovi za kolekciju `live_protocols`, 4 deterministička stanja (`BASELINE_ONLY`, `INGESTED_UNVERIFIED`, `VERIFIED_ACTIVE`, `REJECTED_QUARANTINE`) i 4 domenske invarijante pri upisu.
- [`src/types/projection.types.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/types/projection.types.ts) — Tipovi za `aggregates/current_snapshot`, D'Hondtovih 250 mandata (3 vektora), 5 koalicionih scenarija, Banzhafov indeks i operativnu brzinu unosa.
- [`src/lib/db/sqlite-singleton.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/db/sqlite-singleton.ts) — In-Memory Read-Only Singleton (`node:sqlite` `DatabaseSync`) sa $O(1)$ mapama (`byStationId`, `byStratumId`, `byOpstina`, `byOpstinaAndBm`) i predračunatim Neymanovim težinama.

---

## Faza 2: Jezgro (Matematički motor — Hybrid Neyman-SVD & D'Hondt) — ZAVRŠENO

### Kreirani i ažurirani fajlovi:
- [`src/lib/engine/neyman-stratification.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/engine/neyman-stratification.ts) — Stratifikovani matched-pair swing po 9 strata, Cold Start Bajesovsko skupljanje $\alpha_h = \min(1, n_h / N_{\text{prag}, h})$, sidrenje 31 novog biračkog mjesta iz 2023. godine, strateške Neymanove težine ($w_h^{\text{Voters}}$) i faktor dizajna klastera (`DEFF = 2.5`).
- [`src/lib/engine/svd-imputer.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/engine/svd-imputer.ts) — Algebarska i Mahalanobisova validacija novih zapisnika, Tikhonovljeva (Ridge) OLS regresija nad `svd_dim_1..5` (6x6 sistem sa Gausovom eliminacijom i 5x5 inverznom Gramovom matricom kovarijanse) i sinteza na svih 8.273 biračka mjesta.
- [`src/lib/engine/dhondt-allocator.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/engine/dhondt-allocator.ts) — Deterministički D'Hondtov alokator za 250 poslaničkih mjesta sa 3 paralelna vektora (`pessimistic`, `central`, `optimistic`), praćenjem cenzusa od 3,00% i prirodnog praga manjina, tabelom graničnih količnika (245..255) i kalkulatorom cijene narednog mandata za NADA / Novi DSS.
- [`src/lib/engine/coalition-evaluator.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/engine/coalition-evaluator.ts) — Evaluacija 5 koalicionih scenarija vlasti ($\ge 126$ mandata) i egzaktni Banzhafov indeks moći ($2^7 = 128$ koalicija).
- [`src/lib/engine/velocity-monitor.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/engine/velocity-monitor.ts) — Praćenje urbano/ruralne divergencije brzine unosa u odnosu na `speed_percentile` iz 2023. godine, Z-score alarm (> 2 SD) i detekcija "nijemih biračkih mjesta" (`speed_percentile < 0.30`).
- [`src/lib/engine/index.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/engine/index.ts) — Centralni orkestrator `computeElectionNightSnapshot()` i `buildLiveProtocolDocument()`.
- [`scripts/test-phase2-engine.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/scripts/test-phase2-engine.ts) — Funkcionalni integracioni testovi za Fazu 1 i Fazu 2 (0 grešaka; verifikovan Cold Start na 150 ruralnih BM i Threshold Cliff Edge `0 / 7 / 8` mandata za `ZAV_DVERI` na `3,01%`).

---

## Faza 3: Sinhronizacija (Firebase integracija i Next.js App Router API rute) — ZAVRŠENO

### Kreirani i ažurirani fajlovi:
- [`src/lib/db/firebase-admin.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/db/firebase-admin.ts) — Serverski Firestore konektor za paralelni transakcioni upis u `live_protocols/{station_id}` i 5-sekundni Debounced Batching pri ažuriranju `aggregates/current_snapshot` dokumenta (zaštita od Write Hotspot-a).
- [`src/lib/db/firebase-client.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/db/firebase-client.ts) — Klijentski konektor za pasivno praćenje `current_snapshot` dokumenta u realnom vremenu (`onSnapshot` WebSocket + reaktivni HTTP fallback).
- [`src/app/api/baseline/init/route.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/app/api/baseline/init/route.ts) — Ruta 1 (`GET /api/baseline/init`) za isporuku statičkog manifesta (8.273 BM, 192 opštine, 9 strata).
- [`src/app/api/ingest/protocol/route.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/app/api/ingest/protocol/route.ts) — Ruta 2 (`POST /api/ingest/protocol` i `GET /api/ingest/protocol`) za prijem, unakrsnu validaciju, karantin (HTTP 422), sporni red (`DISPUTED`) i simulaciju izborne noći.
- [`src/app/api/analytics/projection/route.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/app/api/analytics/projection/route.ts) — Ruta 3a (`GET /api/analytics/projection`) za čitanje ili re-kalkulaciju `CurrentSnapshotDocument`.
- [`src/app/api/analytics/recompute/route.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/app/api/analytics/recompute/route.ts) — Ruta 3b (`POST /api/analytics/recompute`) — interni okidač za petosekundno agregiranje stanja i ažuriranje Firestore snimka.
- [`src/app/api/operations/strata-velocity/route.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/app/api/operations/strata-velocity/route.ts) — Ruta 4 (`GET /api/operations/strata-velocity`) za dijagnostiku brzine pristizanja zapisnika po 192 opštine i 9 strata.
- [`scripts/test-phase3-api-sync.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/scripts/test-phase3-api-sync.ts) — Funkcionalni integracioni testovi svih API ruta i Debounced Batching mehanizma (0 grešaka; redukcija Firestore upisa 50x tokom burst naleta).

---

## Faza 4: Prezentacija (Klijentski reaktivni Store i 4 vizuelna panela) — ZAVRŠENO

### Kreirani i ažurirani fajlovi:
- [`src/store/election-night.store.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/store/election-night.store.ts) — Centralni reaktivni klijentski store (`ElectionNightClientStore` + `useElectionNightStore`) koji spaja SQLite manifest (8.273 BM) sa živim Firestore snapshotima i kontrolnim akcijama štaba.
- [`src/components/panels/NationalOverviewPanel/index.tsx`](file:///c:/Users/ognje/Desktop/IzboriApp/src/components/panels/NationalOverviewPanel/index.tsx) — **Panel 1:** Dvostruke trake napretka (Sirovi uzorak vs Projekcija), kontrola cenzusa od 3,00%, zona rizika (2,60% – 3,40% sa 95% CI), prirodni prag manjina i brojač rasutih glasova.
- [`src/components/panels/MandateProjectionPanel/index.tsx`](file:///c:/Users/ognje/Desktop/IzboriApp/src/components/panels/MandateProjectionPanel/index.tsx) — **Panel 2:** Polukružni SVG prikaz Narodne skupštine (250 mjesta), 3 paralelna D'Hondtova vektora (Pesimistični / Centralni / Optimistični), tabela graničnih količnika (245..255), kalkulator cijene mandata za NADA / Novi DSS i regionalni swing po 9 strata.
- [`src/components/panels/CoalitionMatrixPanel/index.tsx`](file:///c:/Users/ognje/Desktop/IzboriApp/src/components/panels/CoalitionMatrixPanel/index.tsx) — **Panel 3:** Interaktivna matrica 5 scenarija vlasti ($\ge 126$ mandata), Kingmaker indikator za NADA / Novi DSS i egzaktni Banzhafov indeks koalicione moći.
- [`src/components/panels/OperationalVelocityPanel/index.tsx`](file:///c:/Users/ognje/Desktop/IzboriApp/src/components/panels/OperationalVelocityPanel/index.tsx) — **Panel 4:** Kriva divergencije urbano/ruralno sa upozorenjem (> 2 SD), tabela "nijemih biračkih mjesta" (`speed_percentile < 0.30`), kartografski WGS84 prikaz 8.273 biračka mjesta i štabni panel za verifikaciju spornih (`DISPUTED`) zapisnika.
- [`src/app/page.tsx`](file:///c:/Users/ognje/Desktop/IzboriApp/src/app/page.tsx), [`src/app/layout.tsx`](file:///c:/Users/ognje/Desktop/IzboriApp/src/app/layout.tsx), [`src/app/globals.css`](file:///c:/Users/ognje/Desktop/IzboriApp/src/app/globals.css), [`package.json`](file:///c:/Users/ognje/Desktop/IzboriApp/package.json), [`tsconfig.json`](file:///c:/Users/ognje/Desktop/IzboriApp/tsconfig.json), [`next.config.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/next.config.ts) — Kompletna Next.js App Router aplikacija.
- [`scripts/test-phase4-store-panels.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/scripts/test-phase4-store-panels.ts) — Funkcionalni end-to-end testovi za Fazu 4.

---

## Faza 5: Instalacija zavisnosti, TypeScript verifikacija i produkcioni Next.js Build — ZAVRŠENO

### Izvršene izmjene i verifikacija:
- Instalirani produkcioni i razvojni paketi (`next@15.5.28`, `react@19`, `react-dom@19`, `firebase`, `firebase-admin`, `typescript@5.8`).
- Otklonjene JSX sintaksne neusklađenosti za matematičke oznake ($\lambda$, $\alpha_{\text{global}}$, $w_h^{\text{Voters}}$, $\sum \beta_i$).
- **Rezultat testiranja:**
  - `npx tsc --noEmit` — **0 grešaka**.
  - `npm test` — **0 grešaka**.
  - `npm run build` (`next build`) — **0 grešaka**.

---

## Faza 6: Forenzički inspektor svih 192 opštine i 8.273 biračka mjesta — ZAVRŠENO

### Kreirani i ažurirani fajlovi:
- [`src/lib/engine/municipality-inspector.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/engine/municipality-inspector.ts) — Analitički modul za:
  - `inspectStationById()`: Istorijski i živi profil svakog BM, 5D SVD vektor, Mahalanobisova distanca i detekcija kašnjenja (`speed_percentile < 0.30`).
  - `inspectMunicipality()`: Dubinski izvještaj sa **Matched-Pair** lokalnim swingom.
  - `listAllMunicipalitiesOverview()`: Rangiranje 192 opštine po NADA swingu, procentu obrade i nijemim biračkim mestima.
- [`src/app/api/operations/municipality/route.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/app/api/operations/municipality/route.ts) — Next.js API ruta (`GET /api/operations/municipality`).
- [`scripts/test-phase6-municipality-inspector.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/scripts/test-phase6-municipality-inspector.ts) — Funkcionalni testovi za Fazu 6 (**0 grešaka**).

---

## Faza 7: Stohastički Monte Carlo simulator (1.000 iteracija) i taktički „Šta-Ako“ (What-If) simulator — ZAVRŠENO

### Kreirani i ažurirani fajlovi:
- [`src/lib/engine/monte-carlo-whatif.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/engine/monte-carlo-whatif.ts) — 1.000 Monte Carlo simulacija (Mulberry32 + Box-Muller) i taktički What-If slajderi sa trenutnim proračunom mandata.
- [`src/app/api/analytics/what-if/route.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/app/api/analytics/what-if/route.ts) — Next.js API ruta (`GET / POST /api/analytics/what-if`).
- [`scripts/test-phase7-monte-carlo-whatif.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/scripts/test-phase7-monte-carlo-whatif.ts) — Funkcionalni testovi za Fazu 7 (**0 grešaka**).

---

## Faza 8: Forenzička detekcija anomalija po listama, karantin registar (`REJECTED_QUARANTINE`), Audit Log štaba i masovni CSV uvoz — ZAVRŠENO

### Kreirani i ažurirani fajlovi:
- [`src/lib/engine/audit-and-batch-csv.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/engine/audit-and-batch-csv.ts) —
  - `analyzeProtocolForensics()`: Z-score odstupanja za 8 lista u odnosu na bazni profil biračkog mesta.
  - `recordAuditLogEntry()` / `getAuditLogEntries()`: Nepromenjivi revizioni dnevnik štaba za sve statusne promene.
  - `parseAndValidateBatchCsvProtocols()`: Masovni paketni unos zapisnika iz CSV fajla.
- [`src/app/api/operations/audit/route.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/app/api/operations/audit/route.ts) — Next.js API ruta (`GET / POST /api/operations/audit`).
- [`scripts/test-phase8-audit-batch-csv.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/scripts/test-phase8-audit-batch-csv.ts) — Funkcionalni testovi za Fazu 8 (**0 grešaka**).

---

## Faza 9: Hronološki Time-Series Replay i Zvanični HQ Izvozni Izvještaji — ZAVRŠENO

### Kreirani i ažurirani fajlovi:
- [`src/lib/engine/timeline-replay-export.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/engine/timeline-replay-export.ts) —
  - `runElectionNightTimelineReplay()`: Hronološka simulacija konvergencije po kontrolnim tačkama (2%, 5%, 10%, 20%, 35%, 50%, 65%, 80%, 95%, 100%) sa proračunom redukcije greške (RMSE) u ranoj noći.
  - `generateNationalSummaryCsv()`: Zvanični nacionalni CSV tabelarni izvoz za štab sa svim listama, 95% CI intervalima, 3 D'Hondtova vektora mandata i Banzhafovim indeksom.
  - `generateStrataBreakdownCsv()`: CSV izvoz po 9 makro-strata sa Neymanovim težinama i regionalnim swingom.
  - `generateMunicipalitiesBreakdownCsv()`: CSV izvoz za svih 192 opština u Srbiji sa Matched-Pair lokalnim swingom.
  - `generateHeadquartersBriefingMarkdown()`: Automatski generator izvršnog strateškog brifinga za rukovodstvo koalicije NADA / Novi DSS u Markdown formatu.

---

## Faza 10: Human-Crafted Editorial Redizajn i Produkciono Pokretanje — ZAVRŠENO

### Izvršene izmjene i poboljšanja korisničkog interfejsa:
- [`src/app/globals.css`](file:///c:/Users/ognje/Desktop/IzboriApp/src/app/globals.css) —
  - Uvedena moderna **Plus Jakarta Sans** tipografija i tabularni brojevi (`tabular-nums`) za sve procente, glasove i mandate.
  - Zamenjena oštra crna pozadina slojevitom tamnom ponoćnom paletom (`#0a0e17` platno, `#111726` kartice, `#161e31` podkartice) sa staklenim ivicama i suptilnim senkama.
  - Uklonjeni veštački AI šabloni (svi-kapiteli, nerealni neonski kontrasti, raw kodni identifikatori).
- [`src/app/page.tsx`](file:///c:/Users/ognje/Desktop/IzboriApp/src/app/page.tsx) —
  - **Novinarski analitički topbar:** Animirani zeleni pulsajući live indikator (`Izborna noć 2026 • Analitički desk`).
  - **Simulacija toka izborne noći:** Realistični tasteri za vremenske prekretnice (`20:30h — Rani ruralni uzorak`, `21:45h — Reprezentativni uzorak`, `23:15h — Završna projekcija`).
  - **Merač napretka obrade:** Dinamička traka prebrojavanja (npr. `24,2% (2.000 / 8.273 BM)`) sa prikazom procenjene izlaznosti (`58,7%`) i važećih glasova (`~3,71M`).
  - **Čitljiva stanja mašine:** `Čeka zapisnik (Projekcija)`, `Potvrđeno i u modelu`, `Sporno / Zahteva proveru`, `Karantin / Greška sume`.
  - **Segmentisana navigacija:** Elegantni tabovi (`Sveobuhvatni War-Room`, `Nacionalni udeli & Cenzus`, `Mandati Skupštine`, `Koalicije & Većina`, `Karta Srbije & Brzina`).
- Panel komponente ([`NationalOverviewPanel`](file:///c:/Users/ognje/Desktop/IzboriApp/src/components/panels/NationalOverviewPanel/index.tsx), [`MandateProjectionPanel`](file:///c:/Users/ognje/Desktop/IzboriApp/src/components/panels/MandateProjectionPanel/index.tsx), [`CoalitionMatrixPanel`](file:///c:/Users/ognje/Desktop/IzboriApp/src/components/panels/CoalitionMatrixPanel/index.tsx), [`OperationalVelocityPanel`](file:///c:/Users/ognje/Desktop/IzboriApp/src/components/panels/OperationalVelocityPanel/index.tsx)) osvežene čistim srpskim opisnim nazivima, legendama i jasnim bedževima (`✓ Sigurno prelazi`, `⚠️ Zona rizika`, `✕ Ispod cenzusa`).

---

## Milestone 1 (2026 Refactoring): SQLite WAL Migration & Dynamic 2026 Schema — ZAVRŠENO

### Kreirani i ažurirani fajlovi:
- [`src/lib/db/sqlite-singleton.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/db/sqlite-singleton.ts) —
  - Prelazak sa read-only na perzistentni **Read-Write WAL** režim (`getSqliteDb`, `getDatabaseConnection`).
  - Konfigurisani SQLite pragmati pri inicijalizaciji konekcije: `journal_mode = WAL`, `synchronous = NORMAL`, `foreign_keys = ON`, `busy_timeout = 5000`.
  - Izvezeni atomski transakcioni omotači (`runInTransaction`, `executeInTransaction`, `withTransaction`) sa `BEGIN IMMEDIATE` / `COMMIT` / `ROLLBACK` i podrškom za ugnježdene `SAVEPOINT` transakcije.
  - Uklonjene sve Firebase/Firestore zavisnosti iz sloja perzistencije.
- [`src/lib/db/schema-2026.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/db/schema-2026.ts) —
  - Tipski ugovori i idempotentna DDL inicijalizacija (`initializeSchema2026`) za 5 operativnih tabela (dok `election_master_baseline` ostaje 100% netaknuta i nemutabilna):
    1. `election_lists_2026` (`list_id`, `ballot_number`, `list_code`, `display_name`, `is_minority_list`, `historical_mapping_json`)
    2. `station_crosswalk_2026` (`station_id`, `opstina_2026`, `broj_bm_2026`, `registered_voters_2026`, `mapping_status`, `UNIQUE(opstina_2026, broj_bm_2026)`)
    3. `sentinel_sample_cohort` (`station_id`, `stratum_id`, `leverage_score`, `speed_percentile`, `composite_utility_score`, `rank_overall`, `is_in_optimal_k`)
    4. `live_station_reports` (`station_id`, `workflow_state`, `reported_turnout`, `valid_ballots`, `invalid_ballots`, `payload_sha256`, `quarantine_reason`, `updated_at_iso`)
    5. `live_list_votes` (`station_id`, `list_id`, `vote_count >= 0`, kompozitni PK `(station_id, list_id)` sa `ON DELETE CASCADE`)
- [`src/lib/db/repository-2026.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/db/repository-2026.ts) —
  - Transakcione repozitorijumske metode: `upsertElectionList2026`, `seedOrUpdateElectionLists2026`, `populateStationCrosswalk2026FromBaselineIfEmpty`, `upsertStationReportWithVotes`, `getVerifiedActiveStationReportsWithVotes`.
- [`src/lib/db/__tests__/schema-2026.test.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/db/__tests__/schema-2026.test.ts) —
  - Jedinični i integracioni testovi za Milestone 1 (**0 grešaka**; verifikovani WAL pragmati, nemutabilnost `election_master_baseline`, mapirane i zero-baseline liste, atomski upsert i transakcioni rollback pri kršenju `CHECK` i `FOREIGN KEY` ograničenja).

---

## Milestone 2 (2026 Refactoring): Sentinel Station Optimizer (Accuracy vs. Speed Pareto Solver) — ZAVRŠENO

### Kreirani i ažurirani fajlovi:
- [`src/lib/engine/sentinel-optimizer.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/engine/sentinel-optimizer.ts) —
  - Deterministički matematički optimizator sentinelnog uzorka nad svih 8.273 biračka mjesta:
    - **Korak A (`computeHatMatrixLeverage`):** Konstrukcija 6-elementnog vektora $x_i = [1, \text{svd\_dim\_1}, \dots, \text{svd\_dim\_5}]$ ponderisanog sa $\sqrt{\text{reg\_2023}_i}$, formiranje regularizovane informacione matrice $M = X^T W X + \lambda I$ ($\lambda = 10^{-6}$), inverzija $M^{-1}$ i normalizacija statističke poluge $h_i = x_i^T M^{-1} x_i$ na $[0, 1]$.
    - **Korak B & C (`rankStationsByStratumBalancedUtility`):** Eksponencijalno diskontovana korisnost po brzini $\text{utility}_i = \text{normalized\_leverage}_i \cdot \exp(-\alpha \cdot \text{speed\_percentile}_i)$ ($\alpha = 2.5$) i proporcionalno rangiranje po svih 9 makro-strata (`rank_overall` $1 \dots 8.273$) prema nacionalnim kvotama biračkog tijela.
    - **Korak D (`evaluateParetoFrontier`):** Evaluacija Pareto granice za $K \in [50, 75, 100, 125, 150, 200, 250, 300, 400, 500, 600, 800, 1000]$ pomoću $6 \times 6$ Tikhonovljeve (Ridge) regresije nad istorijskim baznim listama i automatski izbor optimalnog $K^* = 200$ (`rmse_pct = 0.3340% <= 0.35%`, `max_abs_err_pct = 0.7389%`, `p90_speed_percentile = 0.4736`).
- [`src/lib/db/repository-2026.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/db/repository-2026.ts) —
  - Dodate metode za rad sa `sentinel_sample_cohort`: `saveSentinelSampleCohort`, `getOptimalSentinelCohort`, `getAllSentinelSampleCohortRows` i `getSentinelCohortSummary`.
- [`src/lib/engine/__tests__/sentinel-optimizer.test.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/engine/__tests__/sentinel-optimizer.test.ts) —
  - Jedinični i integracioni testovi za Milestone 2 (**0 grešaka**; verifikovano rangiranje 1..8.273 bez duplikata, proporcionalna zastupljenost svih 9 makro-strata, preciznost `rmse_pct = 0.3340% <= 0.35%` na $K^* = 200$ i upis u `sentinel_sample_cohort`).

---

## Milestone 3 (2026 Refactoring): Dynamic 2026 List & Zero-Baseline Hybrid Projection Engine — ZAVRŠENO

### Kreirani i ažurirani fajlovi:
- [`src/lib/engine/dynamic-projection-2026.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/engine/dynamic-projection-2026.ts) —
  - Deterministički hibridni projekcioni motor za dinamičke izborne liste iz `election_lists_2026`:
    - **Korak A (`classifyProjectionRegime2026`):** Mašina stanja režima projekcije prema broju verifikovanih (`VERIFIED_ACTIVE`) biračkih mjesta ($n_{\text{obs}}$ od $N = 8.273$): `REGIME_0_PRIOR` ($n_{\text{obs}} = 0$), `REGIME_1_COLD_START` ($1 \le n_{\text{obs}} \le 29$), `REGIME_2_SVD_HYBRID` ($30 \le n_{\text{obs}} \le 0,85 \cdot N$) i `REGIME_3_ASYMPTOTIC_LOCK` ($n_{\text{obs}} > 0,85 \cdot N$).
    - **Korak B (`resolveStationBaselineShare2026`):** Parsiranje `historical_mapping_json` i izračunavanje sintetičkog baznog udjela $b_{i,L} \in [0, 1]$ za mapirane koalicije (2023 i 2022 kolone), odnosno $0$ za Zero-Baseline ("wild card") liste (`historical_mapping_json = null`).
    - **Korak C (`runDynamicProjection2026`):** Imputacija neobrađenih biračkih mjesta:
      1. Očekivani važeći listići na osnovu `registered_voters_2026` i Bajesovski skupljene stope važeće izlaznosti u strati ($\alpha_h = \min(1, n_{\text{obs}, h} / 10)$).
      2. Za mapirane liste: upareni lokalni swing u strati $\delta_{h,L}$ ($\alpha_h = \min(1, n_{\text{obs}, h} / 15)$) i $6 \times 6$ ponderisana Tikhonovljeva (Ridge) regresija nad rezidualima $r_{i,L} = y_{i,L} - b_{i,L}$ sa blendom `0.70 * svd + 0.30 * delta` u `REGIME_2_SVD_HYBRID` i `0.30 * svd + 0.70 * delta` u `REGIME_3_ASYMPTOTIC_LOCK`.
      3. Za Zero-Baseline liste (`historical_mapping_json = null`): `0` u `REGIME_0_PRIOR`, IPRW procjena ($w_i = \text{registered\_voters\_2026}_i \cdot \exp(1,5 \cdot \text{speed\_percentile}_i)$) u `REGIME_1_COLD_START`, te direktna $6 \times 6$ SVD Ridge regresija nad opserviranim udjelima $y_{i,L}$ u Režimima 2 i 3.
      4. Normalizacija udjela na svakom neobrađenom BM na sumu $1,0$ i agregacija na nacionalnom nivou i po svih 9 makro-strata.
    - **Korak D (`computeDynamicDHondtMandateAllocation2026` & Sentinel Sub-Projection):** 95% intervali povjerenja, dinamička raspodjela 250 D'Hondt mandata (Pesimistični, Centralni, Optimistični vektor; cenzus 3,00% za opšte liste i prirodni prag za manjinske liste `is_minority_list = 1`) i paralelna Sentinel-Cohort projekcija (`reported_sentinel_stations`, `total_optimal_k`, `sentinel_completion_pct`).
- [`src/lib/engine/index.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/engine/index.ts) —
  - Izvezeni svi javni tipovi i funkcije iz `src/lib/engine/dynamic-projection-2026.ts`.
- [`src/lib/engine/__tests__/dynamic-projection-2026.test.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/engine/__tests__/dynamic-projection-2026.test.ts) —
  - Jedinični i integracioni testovi za Milestone 3 (**0 grešaka**; verifikovana sva 4 režima, IPRW u hladnom startu, rekuperacija nacionalnih udjela na $K^* = 200$ sentinelnih BM sa maksimalnom apsolutnom greškom `0.0156 pp <= 0.50 pp` za mapirane i Zero-Baseline Studentsku listu, D'Hondt raspodjela 250 mandata i čišćenje testnih zapisa).

---

## Milestone 4 (2026 Refactoring): RIK Live Polling Adapter & Idempotent 2026 Ingestion Pipeline — ZAVRŠENO

### Kreirani i ažurirani fajlovi:
- [`src/lib/ingest/rik-poller-2026.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/ingest/rik-poller-2026.ts) —
  - Deterministički, idempotentni cjevovod za prijem, sortiranje i validaciju izbornih zapisnika u realnom vremenu:
    - **Korak A (`transliterateCyrillicToLatin`, `transliterateLatinToCyrillic`, `normalizeSerbianScriptCanonical`, `resolveStationEntity2026`):** Dvosmjerna ćirilično-latinična transliteracija i kanonska normalizacija (digrafi `Lj/Љ`, `Nj/Њ`, `Dž/Dz/Џ`, `Đ/Dj/Ђ`, uklanjanje dijakritika i interpunkcije) i razrješavanje `(opstina, broj_bm)` ili direktnog `station_id` u kanonski zapis iz `station_crosswalk_2026`.
    - **Korak B (`matchAndSortStationVotes2026`):** Dinamičko uparivanje ulaznih glasova po `ballot_number`, `list_code` ili normalizovanom `display_name` (ćirilica/latinica) prema aktivnim listama u `election_lists_2026`, uz podrazumijevanih `0` glasova za izostavljene liste.
    - **Korak C (`computeStationPayloadSha256`, `validateProtocolInvariants2026`, `ingestStationReportsBatch2026`):** Kanonski SHA-256 otisak zapisnika (`payload_sha256`) za idempotentno preskakanje neizmijenjenih zapisnika (`skipped_unchanged`) i provjera 4 algebarske invarijante (`INVALID_NON_NEGATIVE_INTEGER`, `VOTE_SUM_MISMATCH`, `BALLOT_SUM_EXCEEDS_TURNOUT`, `TURNOUT_EXCEEDS_REGISTERED_CEILING`) sa atomskim upisom u `live_station_reports` (`VERIFIED_ACTIVE` ili `REJECTED_QUARANTINE`) i `live_list_votes`.
    - **Korak D (`parseJsonStationReports2026`, `parseCsvStationReports2026`, `parseHtmlTableStationReports2026`, `RikCircuitBreaker`, `pollRikEndpoint2026`):** Višeformatni parseri (JSON nizovi, CSV tabele, HTML `<table>` zapisnici) i 3-stanični Circuit Breaker (`CLOSED`, `OPEN`, `HALF_OPEN`) sa eksponencijalnim backoff-om i jitterom.
- [`src/app/api/ingest/2026/route.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/app/api/ingest/2026/route.ts) —
  - Next.js App Router API ruta (`GET /api/ingest/2026` i `POST /api/ingest/2026`) za status unosa, stanje Circuit Breaker-a, transakcioni prijem JSON/CSV/HTML paketa i automatsko osvježavanje `runDynamicProjection2026` projekcije.
- [`src/lib/engine/index.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/engine/index.ts) —
  - Izvezeni svi javni tipovi i funkcije iz `src/lib/ingest/rik-poller-2026.ts`.
- [`src/lib/ingest/__tests__/rik-poller-2026.test.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/ingest/__tests__/rik-poller-2026.test.ts) —
  - Jedinični i integracioni testovi za Milestone 4 (**0 grešaka**; verifikovana ćirilično/latinična normalizacija opština, sortiranje lista, 100% SHA-256 idempotentnost, izolacija `REJECTED_QUARANTINE` zapisnika od `runDynamicProjection2026`, CSV/HTML parseri, Circuit Breaker i API ruta `/api/ingest/2026`).

---

## Milestone 5: Odvajanje komandne table 2026, Sentinel Cohort Radar i Kontrolni centar unosa u realnom vremenu — ZAVRŠENO

### Kreirani i ažurirani fajlovi:
- [`src/app/api/operations/sentinel-2026/route.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/app/api/operations/sentinel-2026/route.ts) —
  - **`GET /api/operations/sentinel-2026`**:
    - Automatska inicijalizacija `initializeSchema2026` i popunjavanje `station_crosswalk_2026` iz `election_master_baseline`.
    - Automatsko pokretanje Pareto optimizatora (`optimizeAndPersistSentinelSampleCohort`) ukoliko je `sentinel_sample_cohort` prazna.
    - Automatsko sijanje podrazumevane startne liste 2026 (`DEFAULT_STARTER_SLATE_2026`) sa 7 istorijski mapiranih koalicija, manjinskim savezom i Zero-Baseline novom listom (`STUDENT_LIST_2026`).
    - Vraća kompletan operativni snimak: Pareto frontier tačke ($K$ vs RMSE vs P90 brzina, sa $K^* = 200$), realizaciju kohorte po 9 makro-strata, listu prioritetnih sentinel stanica sa živim `workflow_state`, karantin dijagnostiku sa primarnim kodovima grešaka, i najnoviji `runDynamicProjection2026` snimak.
  - **`POST /api/operations/sentinel-2026`**:
    - `CONFIGURE_LISTS_2026`: Dinamička zamjena/ažuriranje izbornih lista u `election_lists_2026`.
    - `SIMULATE_SENTINEL_WAVE`: Simulacija talasa izborne noći (Cold Start 15 BM, K*=200 Sentinel kohorta, Asymptotic Lock 7.200 BM) sa generisanjem glasova za mapirane i zero-baseline liste i kontrolisanim karantinom.
    - `RESET_LIVE_REPORTS`: Trenutni povratak u `REGIME_0_PRIOR` sa 0 redova u `live_station_reports` i `live_list_votes`.
- [`src/lib/db/schema-2026.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/lib/db/schema-2026.ts) —
  - Izvezen `DEFAULT_STARTER_SLATE_2026` i operativni tipovi (`SentinelStratumCompletion2026`, `TopSentinelStationStatus2026`, `QuarantinedStationDiagnostic2026`).
- [`src/store/election-night.store.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/store/election-night.store.ts) —
  - Povezan na `/api/operations/sentinel-2026` i `/api/ingest/2026` sa podesivim intervalom automatskog osvježavanja (`autoRefresh2026Enabled`, `autoRefresh2026IntervalMs`).
  - Izložene akcije: `simulateSentinelWave2026`, `resetLiveReports2026`, `configureLists2026`, i `submitMultiFormatIngest2026`.
- [`src/components/panels/NationalOverviewPanel/index.tsx`](file:///c:/Users/ognje/Desktop/IzboriApp/src/components/panels/NationalOverviewPanel/index.tsx) —
  - Potpuno dinamičko renderovanje lista iz `election_lists_2026` uz vizuelne oznake (Zero-Baseline vs Mapped vs Manjina).
  - Uporedni prikaz: **Full-Stream projekcija** vs **Sentinel-Cohort ($K^* = 200$) sub-projekcija** sa dvostrukim trakama napretka i intervalima poverenja 95% (pesimistički, centralni, optimistički).
- [`src/components/panels/MandateProjectionPanel/index.tsx`](file:///c:/Users/ognje/Desktop/IzboriApp/src/components/panels/MandateProjectionPanel/index.tsx) —
  - Dinamički polukružni SVG Skupštine (250 mandata), dinamički granični količnici (245–255) i uporedni mandati (Full Stream vs Sentinel Cohort).
- [`src/components/panels/CoalitionMatrixPanel/index.tsx`](file:///c:/Users/ognje/Desktop/IzboriApp/src/components/panels/CoalitionMatrixPanel/index.tsx) —
  - Dinamički Banzhafov indeks koalicione moći i evaluacija većine $\ge 126$ mandata sa listama 2026. (uključujući Student listu i NADA / Novi DSS).
- [`src/components/panels/SentinelRadarControlPanel/index.tsx`](file:///c:/Users/ognje/Desktop/IzboriApp/src/components/panels/SentinelRadarControlPanel/index.tsx) —
  - Interaktivna kontrolna traka simulacije talasa (Cold Start, K*=200 Sentinel, 85% Asymptotic Lock, Reset).
  - Tabela Pareto granice ($K$ vs RMSE vs P90 brzina) sa isticanjem optimalnog $K^* = 200$.
  - Praćenje sentinel realizacije po 9 makro-strata sa trakama napretka.
  - Višeformatni unos u realnom vremenu (CSV, JSON, HTML) sa pregledom unosa i inspektorom karantiranih zapisnika (`REJECTED_QUARANTINE`).
- [`src/components/panels/OperationalVelocityPanel/index.tsx`](file:///c:/Users/ognje/Desktop/IzboriApp/src/components/panels/OperationalVelocityPanel/index.tsx) —
  - Prikaz živih stanja 2026 (`BASELINE_ONLY`, `INGESTED_UNVERIFIED`, `VERIFIED_ACTIVE`, `REJECTED_QUARANTINE`).
- [`src/app/page.tsx`](file:///c:/Users/ognje/Desktop/IzboriApp/src/app/page.tsx) —
  - Zaglavlje sa indikatorom aktivnog režima (`REGIME_0_PRIOR`, `REGIME_1_COLD_START`, `REGIME_2_SVD_HYBRID`, `REGIME_3_ASYMPTOTIC_LOCK`), brojačem verifikovanih BM, karantin bedžom, sentinel progresom ($K^* = 200$), prekidačem automatskog osvežavanja i tabovima za navigaciju.
- [`src/app/api/operations/sentinel-2026/__tests__/sentinel-2026-route.test.ts`](file:///c:/Users/ognje/Desktop/IzboriApp/src/app/api/operations/sentinel-2026/__tests__/sentinel-2026-route.test.ts) —
  - Integracioni testovi za operativnu rutu (GET auto-init, seeding, simulacija sva 3 režima, rekonfiguracija lista, reset, čišćenje baze).

---

## Verifikacioni status sistema

| Komponenta / Provera | Status | Rezultat |
| :--- | :---: | :--- |
| **TypeScript tipizacija (`npx tsc --noEmit`)** | **PROŠLO** | **0 grešaka** u celom projektu |
| **Milestone 1 testovi (`npm run test:schema2026`)** | **PROŠLO** | **100% testova prošlo** (WAL, 5 tabela, transakcije i rollback) |
| **Milestone 2 testovi (`npm run test:sentinel`)** | **PROŠLO** | **100% testova prošlo** (Pareto front, $K^* = 200$, `RMSE = 0.3340%`, 8.273 BM u `sentinel_sample_cohort`) |
| **Milestone 3 testovi (`npm run test:projection2026`)** | **PROŠLO** | **100% testova prošlo** (Mapirane + Zero-Baseline liste, `max_abs_err = 0.0156 pp <= 0.50 pp`, 250 D'Hondt mandata) |
| **Milestone 4 testovi (`npm run test:ingest2026`)** | **PROŠLO** | **100% testova prošlo** (Ćirilica/Latinica, SHA-256 idempotentnost, Karantin, JSON/CSV/HTML, Circuit Breaker) |
| **Milestone 5 testovi (`npm run test:sentinel2026`)** | **PROŠLO** | **100% testova prošlo** (GET auto-init, simulacija talasa kroz režime 1-3, konfiguracija lista, reset) |
| **Kompletna test svita (`npm test`)** | **PROŠLO** | **100% testova prošlo** (Svih 11 testnih svita: M1, M2, M3, M4, M5 + Faze 2, 3, 4, 6, 7, 8) |
| **Produkcioni build (`npm run build`)** | **PROŠLO** | Svi statički i dinamički App Router moduli uspešno izgrađeni (`next build`) |
| **SQLite referentna baza** | **VERIFIKOVANA** | 8.273 BM, 192 opštine, 9 strata, 5 SVD dimenzija + 5 operativnih tabela za 2026. |

---

## Sažetak celokupnog projekta

Svi planirani mejlstouni (**Milestone 1, Milestone 2, Milestone 3, Milestone 4 i Milestone 5**) su u potpunosti implementirani, integrisani i rigorozno verifikovani:
1. **Milestone 1**: SQLite WAL perzistencija i dinamička 2026 relaciono-operativna šema bez menjanja nepromenljivog `election_master_baseline` skupa (8.273 BM).
2. **Milestone 2**: Sentinel Station Optimizer sa proračunom statističkog leveridža (Hat matrica $6 \times 6$) i Pareto granice brzine unosa, određujući optimalnih $K^* = 200$ prioritetnih sentinel stanica.
3. **Milestone 3**: Dinamički 2026 hibridni projekcioni motor sa 4-staničnom mašinom režima (`REGIME_0_PRIOR`, `REGIME_1_COLD_START`, `REGIME_2_SVD_HYBRID`, `REGIME_3_ASYMPTOTIC_LOCK`), podrškom za istorijski mapirane koalicije i zero-baseline liste, intervalima poverenja 95% i 250 D'Hondt mandata.
4. **Milestone 4**: RIK poler i višekanalni cjevovod za prijem (JSON/CSV/HTML) sa ćirilično-latiničnom transliteracijom opština, dinamičkim uparivanjem lista, SHA-256 deduplikacijom, automatskim karantinom invarijanti i 3-staničnim Circuit Breaker-om.
5. **Milestone 5**: Dekuplovana komandna tabla bez hardkodiranih ključeva stranaka, Sentinel Cohort Radar UI sa Pareto analizom, paralelna Full-Stream vs Sentinel Cohort projekcija, i interaktivni kontrolni centar za simulaciju izbornih talasa i operativni unos zapisnika.



