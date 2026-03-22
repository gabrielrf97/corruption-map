# Market Research

## Inspirations

### br/acc — Bruno Cesar (@brunoclz)

A 20-year-old developer from Sao Paulo who built a tool aggregating 39+ Brazilian public databases (~1TB) into a Neo4j graph database to expose corruption patterns.

- **Stack:** Neo4j 5 Community, FastAPI (Python 3.12+), React 19 + Vite + TypeScript, 45 ETL pipeline modules, Docker Compose
- **Scale:** 219M+ nodes, 97M+ relationships in production
- **Features:** 8 pattern detectors (sanctioned companies still receiving contracts, split contracts below bidding threshold, contract concentration, etc.), probabilistic entity resolution via splink, investigation workbench with PDF export
- **Users:** Investigators, auditors, power users
- **GitHub:** github.com/brunoclz/br-acc (1.6k stars, AGPL-3.0)
- **Website:** bracc.org

**Key insight:** In an early demo, flagged R$89M in potential irregularities tied to a single politician, including R$47M in amendments directed to a municipality where most contracts were linked to companies owned by the politician's family.

**Our differentiation:**
- br/acc uses force-directed graphs, not geographic maps
- br/acc is a power-user tool, not designed for journalists/citizens
- br/acc has no outcome measurement (our Layer 3)
- br/acc has no input→output divergence detection
- br/acc requires running 1TB locally; we use BigQuery

### WorldView — Bilawal Sidhu (@bilawalsidhu)

Former Google Maps Senior PM (6 years on ARCore, Immersive View, 3D mapping) who built a browser-based geospatial intelligence dashboard in ~3 days by orchestrating up to 8 simultaneous AI coding agents.

- **Stack:** Next.js 16, CesiumJS, Tailwind CSS, Zustand, TypeScript, Ollama
- **Data:** 9 layers — flights (ADS-B), satellites (CelesTrak), earthquakes (USGS), disasters (NASA EONET), weather radar, 3,000+ CCTV cameras, news (GDELT + RSS), military events, asteroids
- **All data:** Publicly available OSINT, no classified feeds
- **OSS clone:** github.com/jedijamez567/worldview_oss (MIT license)

**Our inspiration:** Demonstrates that geographic visualization of open-source data is compelling and viral. Architecture model for map-first approach with multiple data layers.

---

## Existing Brazilian Transparency Tools

### Direct Competitors / Related Products

| Tool | URL | Focus | Users | Gap we fill |
|------|-----|-------|-------|-------------|
| **Portal da Transparencia** | portaldatransparencia.gov.br | Official federal transparency portal | General public | Raw data tables, no visualization, no geographic view, no outcome correlation |
| **Ranking dos Politicos** | rankingdospoliticos.com.br | Scores legislators on fiscal responsibility votes | Voters | No geographic/municipal dimension, no spending outcomes |
| **Serenata de Amor / Rosie** | serenata.ai | AI audit of parliamentary CEAP spending | Journalists, activists | Federal spending only, no municipal data, no map, no outcomes |
| **Congresso em Foco** | congressoemfoco.uol.com.br | Legislative journalism + transparency profiles | General public | Journalism platform, not a data exploration tool |
| **Querido Diario** | queridodiario.ok.org.br | Full-text search of municipal official gazettes | Journalists, researchers | Raw text search, no structured analysis, no visualization |
| **Legisla Brasil** | legislabrasil.org | Legislative productivity index | Researchers | Federal legislators only, no municipal connection |
| **Atlas Politico** | atlaspolitico.com.br | Ideological mapping of legislators | Researchers, media | No financial/spending/outcome data |
| **Radar Parlamentar** | radarparlamentar.polignu.org | PCA clustering of deputies by voting patterns | Researchers | Academic tool, no spending/outcome analysis |
| **QEdu** | qedu.org.br | Education data per school/municipality | Educators, researchers | Education only, no financial correlation |
| **Meu Congresso Nacional** | meucongresso.com.br | Simplified legislative activity view | Citizens | Federal only, no municipal dimension |
| **Quem Me Representa** | quemmerepresenta.org.br | Match voters with deputies by policy positions | Voters | Voting alignment only, no spending/outcome data |

### International References

| Tool | Country | Relevance |
|------|---------|-----------|
| **USAspending.gov** | USA | Federal spending visualization with geographic drill-down. Similar concept but US-specific. |
| **OpenSpending** | International | Open data platform for government spending. Community-driven. |
| **Follow the Money** | Netherlands | Tracks subsidies and government spending. Strong visualization. |
| **Subsidietracker** | Netherlands | Subsidy tracking with company-level detail. |

---

## Unique Value Proposition

No existing tool connects these four dimensions geographically:

```
1. MONEY IN     — Where federal money goes (transfers, emendas, convenios)
2. MONEY OUT    — How cities spend it (budget execution, contracts, suppliers)
3. OUTCOMES     — What it produces (health infra, education quality, sanitation)
4. RED FLAGS    — Where things go wrong (legal violations + input→output gaps)
```

### Why this matters for 2026

- Brazil has federal elections in October 2026
- Emendas parlamentares have become increasingly controversial (STF rulings on "emendas pix")
- Public demand for fiscal transparency is high
- No tool lets a voter see: "my city received R$10M from Deputado X, spent it on health, but hospital beds decreased"
- The geographic dimension makes the abstract concrete — money flowing on a map tells stories that spreadsheets cannot

### Target users and their jobs-to-be-done

**Journalist:**
- "I need to find a story about public spending in my region"
- "I need data to support or challenge a politician's claims about investment"
- "I need to compare how different cities in the same state use federal money"

**Citizen:**
- "I want to know where my city's money comes from and where it goes"
- "I want to understand why my city's hospital is underfunded despite receiving federal transfers"
- "I want to make an informed vote in 2026 based on what politicians actually delivered"

### Distribution strategy (not in scope for MVP, but considered)

- Embeddable widgets for news organizations
- Shareable city reports (URL per municipality)
- Social media cards (auto-generated city summaries)
- API for journalists and researchers
- Portuguese-first, English documentation for international interest
