# Project A: City Transparency Map

## Vision

A map-first web application that visualizes how public money flows to and through Brazil's 5,570 municipalities, what it produces, and where things go wrong.

**Users:** Journalists and citizens.
**Core question:** "Where does public money go, and what does it produce?"

---

## Product Design

### Entry Point

User lands on a full-screen map of Brazil. Municipalities are colored by a default metric (federal transfers per capita). Animated arcs show money flowing from Brasilia to state capitals (aggregated), and from state capitals to individual cities.

### Interaction Model: Map-First

```
BRAZIL-WIDE VIEW (zoom level 0)
│
│  Animated arcs flowing FROM Brasilia TO each state capital
│  Arc thickness = total federal transfers to that state
│  Arc color:
│    Green = less than 51% of state's municipalities have critical legal violations
│    Red = 51%+ of state's municipalities have critical legal violations
│
├── CLICK STATE (zoom level 1)
│
│   Animated arcs flowing FROM state capital TO each municipality
│   Arc thickness = transfers to that municipality
│   Arc color:
│     Green = city has 0 critical red flags
│     Yellow = city has medium-severity flags
│     Red = city has critical flags (legal violations or input→output divergence)
│
│   Municipality polygons colored by same logic
│
└── CLICK CITY (zoom level 2)

    Side panel opens with full city profile (4 layers)
    Links out to Portal da Transparencia for that city
```

### Default Map View

Federal transfers per capita — neutral starting point that invites exploration. A dropdown lets users recolor the map by:
- Legal threshold violations (health, education, personnel)
- Health spending compliance
- Education spending compliance
- Input→Output divergence scores

### Arc Color Logic

State-level arc color is based on **critical legal violations only** (hard legal thresholds: LRF personnel limit, constitutional health/education minimums). The threshold is 51%+ of municipalities in that state having at least one critical violation.

### Incomplete Data

Cities with missing data are shown with **grayed-out UI** — visible on the map, but visually distinct. Missing data is flagged explicitly in the city profile. Missing data is itself a story.

---

## Data Architecture: 4 Layers

### Layer 1: Money In

"Where does this city's money come from?"

| Field | Detail Level | Source |
|-------|-------------|--------|
| Total federal transfers (FPM, FUNDEB, ICMS, SUS, etc.) | Aggregate + breakdown by type | Tesouro/SICONFI |
| Emendas parlamentares received | Aggregate + list of each emenda (author, party, type, value, function) | Portal da Transparencia |
| Convenios/agreements | Aggregate + list of each convenio (object, grantor organ, value, status) | TransfereGov |
| Voluntary transfers | Each transfer with source organ, program, value, date | Portal da Transparencia |
| Health transfers (fundo a fundo) | FNS transfers | FNS |
| Education transfers (FUNDEB, PNAE, PDDE) | FNDE transfers | FNDE |
| Year-over-year trend | % change vs previous year | SICONFI |

### Layer 2: Money Out

"How does this city spend it?"

| Field | Detail Level | Source |
|-------|-------------|--------|
| Total budget execution | Aggregate by function | SICONFI (RREO) |
| % spent on personnel | vs 54% LRF limit | SICONFI |
| % spent on health | vs 15% constitutional minimum | SIOPS |
| % spent on education | vs 25% constitutional minimum | SIOPE |
| % spent on security, urbanismo, saneamento, etc. | Each function as % + absolute R$ | SICONFI |
| Which companies received health contracts | Company name, CNPJ, value, contract object | TCE APIs / PNCP |
| Which companies received education contracts | Company name, CNPJ, value, contract object | TCE APIs / PNCP |
| Top suppliers | Full list with CNPJ, total value, number of contracts | TCE APIs / PNCP |
| Each contract | Object, value, modality (licitacao/dispensa/inexigibilidade), supplier CNPJ | PNCP / TCE |

### Layer 3: Outcomes

"What results is the money producing?"

| Field | Detail Level | Source |
|-------|-------------|--------|
| Hospital beds per 10k inhabitants | vs state average, trend over years | CNES/DataSUS |
| Doctors per 10k inhabitants | vs state average, trend | CNES |
| SUS facilities count | Total health establishments | CNES |
| IDEB score (anos iniciais + finais) | vs state average, trend | INEP |
| Schools count | Total | Censo Escolar |
| Teachers count | Total | Censo Escolar |
| Water coverage % | Trend over years | SNIS |
| Sewage coverage % | Trend over years | SNIS |
| Formal employment total | By sector, trend | RAIS/CAGED |
| CGU transparency score | 0-10 scale | CGU EBT |
| Has ombudsman (Y/N) | Governance quality | MUNIC (IBGE) |
| Has transparency portal (Y/N) | Governance quality | MUNIC (IBGE) |

### Layer 4: Red Flags

"What looks wrong?"

Red flags require Layer 3 (outcomes) to exist — you can't say something is wrong without knowing what "right" looks like.

**Baseline methodology (in priority order):**

1. **Legal thresholds (B)** — Hard limits defined by Brazilian law. Objective, not opinion.
2. **Input→Output ratios (C)** — Money in vs measurable outcomes. Requires careful methodology.

(Peer comparison (A) was deprioritized — not as relevant for journalists/citizens.)

| Flag | Logic | Severity |
|------|-------|----------|
| Personnel spending > 54% | SICONFI personnel % > LRF limit (art. 20) | Critical |
| Health spending < 15% | SIOPS % < constitutional minimum (LC 141) | Critical |
| Education spending < 25% | SIOPE % < constitutional minimum (CF art. 212) | Critical |
| Sanctioned companies receiving contracts | CEIS list x PNCP/TCE contracts | High |
| Health spending up, beds/doctors down | Layer 1+2 vs Layer 3 trend divergence | High |
| Education spending up, IDEB down | Layer 1+2 vs Layer 3 trend divergence | High |
| Low transparency score | CGU EBT < 5.0 | Medium |
| Contract concentration | Single supplier > X% of total spending | Medium |

---

## Tech Stack

### Frontend
- **Next.js** — React framework, app router
- **Deck.gl + react-map-gl** — WebGL map rendering (ArcLayer for money flows, GeoJsonLayer for choropleth)
- **MapLibre GL JS** — Free basemap (no token needed)
- **TypeScript**

### Backend / Data
- **Supabase** — PostgreSQL + PostGIS, free tier (500MB DB)
- **Pre-computed city profiles** — ~5,570 rows, ~280MB estimated
- **IBGE TopoJSON** — Municipality boundaries served from CDN

### Data Pipeline
- **Python** (pandas, basedosdados package)
- **BigQuery** (basedosdados.org) — Upstream data source, 1TB/month free
- **GitHub Actions** — Nightly pipeline to refresh Supabase from BigQuery + REST APIs
- Keeps Supabase alive (no cold start after 7 days inactivity)

### Deploy
- **Vercel** — Frontend hosting (free tier)
- **Supabase** — Database (free tier)
- **GitHub Actions** — Data pipeline (2,000 min/month free)
- **Total cost: $0**

---

## Data Sources

### Money In (Federal → Municipal)

| Source | Data | API/URL | Auth | Format |
|--------|------|---------|------|--------|
| Tesouro/SICONFI | Constitutional transfers (FPM, FUNDEB, ICMS) | `apidatalake.tesouro.gov.br/ords/siconfi/` | None | JSON |
| Portal da Transparencia | Emendas, convenios, voluntary transfers | `api.portaldatransparencia.gov.br` | Free key | JSON |
| TransfereGov | Federal agreements, proposals, payments | `api.transferegov.sistema.gov.br` | None | JSON/CSV |
| FNS | Health transfers (fundo a fundo) | `consultafns.saude.gov.br` (panels) | None | Web |
| FNDE | Education transfers (FUNDEB, PNAE, PDDE) | `dados.gov.br` CKAN | None | CSV/JSON |
| basedosdados.org | Pre-normalized emendas, transfers | BigQuery SQL | Google account | SQL |

### Money Out (How cities spend)

| Source | Data | API/URL | Auth | Format |
|--------|------|---------|------|--------|
| SICONFI/FINBRA | Full budget execution by function | `apidatalake.tesouro.gov.br/ords/siconfi/` | None | JSON |
| SIOPE | Education spending breakdown, 25% compliance | `fnde.gov.br/siope` | None | Web/CSV |
| SIOPS | Health spending breakdown, 15% compliance | `siops-consulta-publica-api.saude.gov.br` | None | JSON |
| PNCP | Municipal procurement (contracts, suppliers) | `pncp.gov.br/api/pncp/v1/` | None | JSON |
| TCE-SP | SP municipal expenses with supplier detail | `transparencia.tce.sp.gov.br/api/` | None | JSON |
| TCE-RJ | RJ contracts, bids, winners, payroll, COVID spending | `dados.tcerj.tc.br/api/v1/` | None | JSON |
| TCE-PE | PE expenses, suppliers, bids, contracts | `sistemas.tce.pe.gov.br/DadosAbertos/` | None | XML/JSON |
| TCE-RS | RS empenhos, education/health indices | `dados.tce.rs.gov.br/` (CKAN) | None | Various |
| TCE-SC | SC municipal accounts (e-SFINGE) | `servicos.tcesc.tc.br/endpoints-portal-transparencia/` | None | JSON |
| TCE-MG | MG municipal accounting (SICOM) | `dadosabertos.tce.mg.gov.br/` | None | Various |
| TCE-PB | PB commitments, payroll (SAGRES) | `dados-abertos.tce.pb.gov.br/` | None | Pipe-delimited |

### Outcomes

| Source | Data | API/URL | Auth | Format |
|--------|------|---------|------|--------|
| CNES/DataSUS | Hospitals, beds, doctors per municipality | `apidadosabertos.saude.gov.br/` | None | JSON |
| INEP/IDEB | Education quality scores per municipality | `api.dadosabertosinep.org/v1` | None | JSON |
| Censo Escolar | Schools, teachers count | `basedosdados.br_inep_censo_escolar` | Google account | BigQuery |
| SNIS | Water/sewage coverage % | `app4.mdr.gov.br/serieHistorica/` | None | Web/XLS |
| RAIS/CAGED | Formal employment by sector | `basedosdados.br_me_rais` / `br_me_caged` | Google account | BigQuery |
| MUNIC (IBGE) | Governance survey (transparency portal, ombudsman) | SIDRA API | None | JSON |
| CGU EBT | Transparency compliance score (0-10) | `mbt.cgu.gov.br` | None | Web |

### Red Flags / Accountability

| Source | Data | API/URL | Auth | Format |
|--------|------|---------|------|--------|
| CGU CEIS/CNEP | Sanctioned/punished companies | Portal da Transparencia API | Free key | JSON |
| CGU EBT | Transparency compliance score | `mbt.cgu.gov.br` | None | Web |
| TCU | Irregular accounts, audits | `portal.tcu.gov.br/dados-abertos` | None | CSV/JSON |
| Querido Diario | Full text of 5,000+ municipal gazettes | `queridodiario.ok.org.br/api/` | None | JSON |

### Geography

| Source | Data | API/URL | Auth | Format |
|--------|------|---------|------|--------|
| IBGE Malhas API v3 | GeoJSON/TopoJSON polygons for all municipalities | `servicodados.ibge.gov.br/api/v3/malhas/` | None | GeoJSON/TopoJSON |
| IBGE Localidades API | IBGE codes, names, regions | `servicodados.ibge.gov.br/api/v1/localidades/` | None | JSON |
| IBGE SIDRA | Population, GDP per municipality | `apisidra.ibge.gov.br/` | None | JSON |
| kelvins/municipios-brasileiros | Municipality centroids (lat/lon) | GitHub CSV | None | CSV |
| Atlas Brasil | HDI per municipality | `atlasbrasil.org.br` | None | XLS/CSV |

---

## Key Design Decisions & Tradeoffs

### 1. Data Strategy: BigQuery-first, no custom ETL

**Decision:** Use basedosdados.org (BigQuery) as the upstream data source instead of building custom ETL pipelines for each government API.

**Why:** basedosdados already has pre-normalized tables for emendas, CNPJ, TSE, transfers, sanctions — all with IBGE municipality codes joined. Skips the entire ETL problem for v1.

**Tradeoff:** Cannot do multi-hop relationship traversal (politician → family → companies → contracts), probabilistic entity resolution, temporal ownership tracking, or real-time monitoring. These require owning the pipeline (like br/acc does).

**What this means:** The system answers single-hop lookups and aggregations well. It cannot answer "show me all companies owned by relatives of the politician who authored the emenda." This is acceptable for the journalist/citizen audience.

### 2. No Graph Database

**Decision:** PostgreSQL (Supabase) instead of Neo4j.

**Why:** The core interaction is geographic (map-first), not graph-first. PostGIS gives geographic queries. The relationship queries needed (city → transfers, city → contracts) are simple JOINs, not multi-hop traversals.

**Tradeoff:** If the product evolves toward investigative/auditor use cases requiring multi-hop graph queries, a migration to Neo4j (or adding it alongside) would be needed.

### 3. Legal Thresholds as Red Flags, Not "Corruption Scores"

**Decision:** Red flags are based on hard legal limits (LRF, constitutional minimums) and input→output divergence, not subjective "corruption scores."

**Why:** Legal thresholds are objective and defensible. Calling something "corrupt" without due process is legally risky and editorially irresponsible. The system surfaces facts; users draw conclusions.

**Why Layer 4 requires Layer 3:** You can't say "this smells wrong" without knowing what "right" looks like. A city receiving R$50M in health transfers isn't suspicious until you see it has 2 hospital beds per 10,000 people. Red flags without outcomes is noise.

### 4. Peer Comparison Deprioritized

**Decision:** Not implementing peer comparison (comparing city X to similar cities by population/HDI/region) in the initial version.

**Why:** Legal thresholds and input→output ratios are more directly actionable. Peer comparison adds complexity (defining "similar cities") without proportional user value for journalists/citizens.

### 5. Politicians as Data Field, Not Feature

**Decision:** Politicians appear in city profiles only as emenda authors — a data field, not a core feature.

**Why:** A politician performance platform is a separate product with its own UX. Voters want to evaluate candidates, not navigate a municipality map. Forcing politician profiles into a map-first tool dilutes both products.

**Future:** Project B (Politician Performance Platform) can share the same Supabase database and link to/from Project A. Ideal launch timing: July 2026 when candidate registrations open for federal elections.

### 6. Supabase over Static JSON CDN

**Decision:** Use Supabase free PostgreSQL instead of pre-computed static JSON files.

**Why:** While static JSON is faster (edge-cached, ~20ms), Supabase enables dynamic queries essential for the product:
- "Show me all cities in Para that spend less than 15% on health"
- Filtering/sorting the map by different metrics
- Full-text search on city/company names
- PostGIS geographic queries
- Future growth (user accounts, saved investigations)

The 500MB free tier fits the estimated ~280MB city profiles comfortably. The nightly GitHub Action keeps the database alive (avoids 7-day inactivity cold start).

### 7. Money Flow Animation: Hybrid Approach

**Decision:** Brazil view shows Brasilia → state capitals (aggregated). State view shows direct arcs to each municipality.

**Why:** Federal transfers actually go directly from Brasilia to municipalities (FPM goes straight to city bank accounts). But showing 5,570 arcs from Brasilia is visually overwhelming. The hybrid approach is both truthful at each zoom level and readable.

### 8. TCE Integration is State-by-State

**Decision:** Integrate TCE APIs progressively, starting with the best ones (TCE-SP, TCE-RJ, TCE-PE).

**Why:** Each state audit court has a different API format, different endpoints, different data structures. There is no unified national TCE API. Coverage: ~3,400 municipalities across the TCEs with documented APIs. The remaining ~2,170 will show grayed-out supplier-level data until their TCEs are integrated or PNCP coverage expands.

---

## Market Research: Competitive Landscape

### br/acc (Bruno Cesar)
- **What:** Graph-based tool mapping corruption patterns across 39+ federal databases
- **Stack:** Neo4j, FastAPI, React, 45 ETL pipelines
- **Scale:** 219M nodes, 97M relationships, ~1TB of data
- **Differentiation from us:** Force-directed graph visualization, not geographic. Power-user tool for investigators/auditors, not journalists/citizens. No geographic money flow visualization. No outcome measurement (Layer 3). No input→output divergence detection.
- **GitHub:** github.com/brunoclz/br-acc (1.6k stars, AGPL-3.0)

### WorldView OSS (Bilawal Sidhu inspiration)
- **What:** Geospatial intelligence dashboard on a 3D globe
- **Stack:** Next.js, CesiumJS, Zustand, Ollama
- **Relevance:** Demonstrates that geographic visualization of OSINT data is compelling and viral. Architecture inspiration for map-first approach.
- **Differentiation:** WorldView is global defense/intelligence. We are Brazil municipal transparency.

### Existing Brazilian Tools

| Tool | Focus | Gap we fill |
|------|-------|-------------|
| Ranking dos Politicos | Scores legislators on votes | No geographic/municipal dimension, no outcomes |
| Serenata de Amor / Rosie | AI audit of parliamentary spending (CEAP) | No municipal spending, no map, no outcomes |
| Congresso em Foco | Legislative journalism | Not a data tool, no map |
| Querido Diario | Municipal gazette text search | Raw text, no visualization, no structured analysis |
| Portal da Transparencia | Official government transparency portal | Raw data, no geographic visualization, no outcome correlation |
| Legisla Brasil | Legislative productivity index | Federal only, no municipal connection |
| Atlas Politico | Ideological mapping | No financial/outcome data |

### Our Unique Position

No existing tool connects these four dimensions geographically:
1. Where federal money goes (Money In)
2. How cities spend it (Money Out)
3. What it produces (Outcomes)
4. Where things go wrong (Red Flags)

The closest is br/acc, which has dimensions 1, 2, and partial 4 — but not on a map, not with outcomes, and not for a citizen/journalist audience.

---

## Future: Project B — Politician Performance Platform

Separate product sharing the same data layer. Designed for the 2026 federal elections.

**Target launch:** July 2026 (when candidate registrations open)

**Data available for politicians:**
- Voting record (Camara/Senado API)
- Attendance (Camara/Senado API)
- Bills authored (Camara API)
- CEAP/CEAPS spending with supplier CNPJs (Camara/Senado API)
- Emendas authored — which cities, how much (Portal da Transparencia)
- Campaign donations received (TSE DivulgaCandContas)
- Campaign spending (TSE DivulgaCandContas)
- Asset evolution across elections (TSE bulk data)
- Party switches (TSE filiacao data)
- Ficha limpa status (TSE registration status)
- Caucuses/frentes parlamentares (Camara API)

**Connection to Project A:** Both products query the same Supabase database. Project A can link to Project B ("see this politician's full profile"). Project B can link to Project A ("see the cities this politician sent money to").

---

## MVP Scope

### In MVP
- Brazil → state → city zoom with choropleth coloring
- Static choropleth first (animated arcs deferred)
- Layer 1: Money In (SICONFI federal transfers, emendas list)
- Layer 4: Red Flags (3 legal threshold violations: personnel >54%, health <15%, education <25%)
- City side panel with key metrics
- Single default metric (federal transfers per capita)
- ~20 data fields per city
- BigQuery → Supabase pipeline via GitHub Actions
- IBGE TopoJSON for municipality boundaries

### Post-MVP
- Animated money flow arcs (Deck.gl ArcLayer)
- Layer 2: Money Out (SICONFI budget execution, PNCP/TCE contract data)
- Layer 3: Outcomes (CNES health, IDEB education, SNIS sanitation)
- Layer 4 expansion: Input→Output divergence detection
- Metric dropdown switcher (recolor map by different metrics)
- Full contract/supplier lists per city
- TCE integration (state by state)
- 50+ data fields per city
- Links to Portal da Transparencia
- Querido Diario gazette search per city
