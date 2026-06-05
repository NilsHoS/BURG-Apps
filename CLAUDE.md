# BURG-Apps — Projectcontext voor Claude Code

## Wat is dit project?
Multi-tool recruitment portal voor BURG. Gebouwd door Max (data/backend) en Nils (frontend/app).
Gehost op: https://nilshos.github.io/BURG-Apps/
Lokale repo: C:\Users\MaxvanLeeuwenBURGBed\BURG-Apps

## Stack
- **Frontend**: GitHub Pages (HTML/CSS/JS)
- **Database**: Supabase (project: burg-jobs, id: ziwqshuabwcthqjspuso, regio: eu-west-1)
- **Automatisering**: n8n (mvl1009.app.n8n.cloud)
- **Scraping**: Thunderbit (LinkedIn, Indeed) via Google Sheets → n8n
- **Branch workflow**: Max werkt op `feature/database-koppeling`, Nils reviewt en merget via PR

## Pagina's in de app
- `vacatures.html` — overzicht nieuwe vacatures uit Supabase
- `pipeline.html` — goedgekeurde vacatures met recruiter assignment en statusbeheer
- `swipe` — swipe interface per medewerker (achter inlogscherm per medewerker)
- `fee-calculator` — fee berekening tool
- `placement-distribution` — plaatsing distributie tool

---

## Supabase structuur

### Tabel: `jobs` (hoofdtabel, leeg na fresh start juni 2026)
| Kolom | Type | Toelichting |
|---|---|---|
| id | uuid | primary key |
| job_title | text | |
| company_name | text | |
| job_location | text | |
| job_url | text | |
| job_description | text | volledige vacaturetekst |
| data_source | text | 'linkedin' / 'indeed' / 'werkzoeken.nl' / 'bedrijfswebsite' |
| review_status | text | default 'pending' — swipe beslissing |
| review_notes | text | |
| reviewed_by | text | welke recruiter |
| reviewed_at | timestamptz | |
| nogo_reason | text | reden van no-go |
| seniority_level | text | nu altijd 'medior' — nog niet via LLM bepaald |
| seniority_reviewed_at | timestamptz | |
| sales_status | text | |
| sales_notes | text | |
| assigned_to | text | |
| contact_name | text | |
| contact_email | text | |
| contact_phone | text | |
| contact_title | text | |
| contact_linkedin | text | |
| contact_raw | text | |
| recruiter_name | text | LinkedIn recruiter |
| recruiter_linkedin | text | LinkedIn recruiter URL |
| recruiter_headline | text | |
| company_website | text | |
| company_linkedin | text | |
| company_address | text | |
| company_industry | text | |
| company_phone | text | via Apollo enrichment |
| company_name_display | text | |
| enriched_at | timestamptz | Apollo enrichment timestamp |
| employment_type | text | |
| industry | text | |
| salary | text | |
| posted_at | timestamptz | |
| date_scraped | timestamptz | |
| created_at | timestamptz | default now() |
| company_name_normalized | text | voor dedup |
| job_title_normalized | text | voor dedup |

### Tabel: `geziene_vacatures` (dedup tabel)
Kolommen: `job_title_normalized`, `company_name_normalized`, `scraped_at`, `job_url`
Dedup-check via "If row does not exist" node in n8n op `job_url` (voorheen op titel+bedrijf combinatie).

### Tabel: `employees` (6 rijen)
Kolommen: `id`, `name`, `email`, `fte_hours`, `seniority_levels`, `is_present`

---

## n8n Workflows

### Workflow 1: "Apify Workflow 1" (id: R3Ky6mByr4CRooNC) ✅ ACTIEF — omgebouwd naar Thunderbit
**Flow:**
```
Schedule Trigger (07:45)
  → Get row(s) in sheet (Indeed Scraper — Google Sheet)  [parallel]
  → Get row(s) in sheet1 (LinkedIn Scraper — Google Sheet)  [parallel]
  → Merge
  → normalisatie_en_filtering (Code node)
  → If row does not exist (geziene_vacatures check op job_url)
  → Create a row (jobs tabel)
  → Insert row (geziene_vacatures Data Table)
```

**Thunderbit scrapers (vervangen Apify):**
- LinkedIn: Thunderbit scraper, draait dagelijks om 06:15, schrijft naar Google Sheet 'LinkedIn Scraper'
- Indeed: Thunderbit scraper, draait dagelijks om 06:45, schrijft naar Google Sheet 'Indeed Scraper'
- Alle Apify nodes zijn disabled (nog aanwezig in workflow maar inactief)

**normalisatie_en_filtering code node — wat hij doet:**
1. `detectSource(j)` — bepaalt bron op basis van veldnamen:
   - `thunderbit_linkedin`: veld `Functietitel` aanwezig zonder `Functieomschrijving`
   - `thunderbit_indeed`: veld `Functietitel` + `Functieomschrijving` aanwezig
   - `thunderbit_indeed_oud`: veld `Job Title` + `Data Source` met indeed.com
2. `normalizeItem(j)` — mapt ruwe Thunderbit output naar Supabase velden per source
3. `date_scraped` wordt gevuld met `new Date().toISOString()`
4. posted_at 1970-fix toegevoegd (filtert ongeldige epoch-datums)
5. Corrupt items filter toegevoegd
6. Filtert op QHSSE keywords in jobtitel
7. Filtert op excludedCompanies (incl. &deBlauw Search toegevoegd)
8. Filtert op excludedIndustries (staffing, HR, executive search)
9. Filtert op excludedTitleWords (stage, intern, stagiair)
10. Deduplicatie binnen batch op job_url
11. Output: genormaliseerde items met job_title_normalized en company_name_normalized

### Workflow 2: "Apollo Enrichment" (id: z297XGfx0k31sB5a) ✅ ACTIEF
**Flow:**
```
Webhook (POST) 
  → Get a row (jobs tabel op job_id) 
  → If (recruiter_linkedin niet leeg) 
  → HTTP Request (Apollo API: people/match op linkedin_url) 
  → Update a row (contact_email, contact_phone, enriched_at)
```
Apollo gebruikt `recruiter_linkedin` als lookup key.
Webhook URL: `https://mvl1009.app.n8n.cloud/webhook/d9c398f5-d635-4fd2-b4ce-231b3bc087fe`

---

## Wat werkt al ✅
- LinkedIn + Indeed scraping via Thunderbit → Google Sheets → n8n (dagelijks)
- Normalisatie + QHSSE keyword filter + excludedCompanies filter + dedup op job_url
- Supabase insert via n8n
- Swipe interface per medewerker (achter inlogscherm)
- Pipeline.html met recruiter assignment en statusbeheer
- Apollo enrichment workflow via webhook
- Volledige Supabase structuur aanwezig

## Wat nog moet gebeuren 🔧

### 1. Werkzoeken.nl scraping herstellen
- Was actief via Apify, nu disabled
- Bepalen of via Thunderbit of andere aanpak

### 2. Bedrijfswebsite scraping toevoegen
- Was gepland via Apify Website Content Crawler
- Aanpak herzien nu Apify disabled is

### 3. Anthropic API koppelen aan n8n (via bedrijfsaccount)
- LLM-based QHSSE classificatie ter vervanging van keyword filter
- Automatische seniority bepaling (nu krijgt alles 'medior')
- Model: claude-haiku voor kostenefficiëntie (~$0.001 per vacature)

### 4. Volledig automatische go/no-go (langetermijndoel)
- LLM beoordeelt vacatures op basis van historische swipe data in jobs tabel
- Elke swipe nu = trainingsdata (review_status + nogo_reason zijn al aanwezig)
- Swipe scherm wordt optioneel zodra model goed genoeg is

---

## Werkwijze
- Max werkt in `C:\Users\MaxvanLeeuwenBURGBed\BURG-Apps`
- Branch: `feature/database-koppeling`
- Wijzigingen via PR naar main — Nils reviewt en merget
- Claude Code starten: `cd C:\Users\MaxvanLeeuwenBURGBed\BURG-Apps` → `claude`
- Nils werkt aan frontend/app features via zijn eigen multi-agent Claude Code setup

---

## Instructie voor Claude Code: CLAUDE.md bijwerken

Werk aan het einde van elke werksessie de CLAUDE.md bij op basis van wat er gedaan is:
- Zet ✅ achter voltooide stappen
- Voeg nieuwe Supabase kolommen toe als die zijn aangemaakt
- Voeg nieuwe n8n workflows of nodes toe als die zijn gebouwd
- Verplaats afgeronde taken van "Wat nog moet gebeuren" naar "Wat werkt al"
- Commit de bijgewerkte CLAUDE.md mee met je andere wijzigingen

Doe dit altijd zonder dat Max erom hoeft te vragen.
