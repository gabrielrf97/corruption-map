# Project B: Politician Performance Platform

## Vision

A search-and-ranking platform that gives Brazilian voters a complete, data-driven profile of every federal deputy and senator — their money trail, legislative performance, and legal standing — in one place, before the October 2026 elections.

**Users:** Voters, journalists, citizens.
**Core question:** "Is this politician worth my vote?"

---

## Product Design

### Entry Point: Ranking

User lands on a sortable leaderboard of all 594 federal legislators (513 deputies + 81 senators). Default sort: attendance %. Columns show key metrics at a glance. User can:
- Sort by any metric (attendance, spending, bills approved, asset growth, emenda volume)
- Filter by party, state, position (deputy/senator), caucus
- Search by name
- Click any politician to open their full profile

### Politician Profile

Full-page profile with tabbed sections:

```
HEADER
├── Photo (TSE)
├── Name, party, state, position
├── Ficha Limpa status badge (green/red)
├── Quick stats row: attendance %, bills approved, CEAP total, emendas total
└── "Compare" button

TABS
├── Money Trail
├── Performance
├── Ficha Limpa & Sanctions
└── History
```

### Side-by-Side Comparison

Pick 2-3 politicians → see all metrics in a comparison table. Shareable via URL. Designed for voters deciding between candidates in the same district.

---

## Data Architecture

### A. Money Trail

#### Campaign Funding (TSE)

| Data | Source | Fields |
|------|--------|--------|
| All donations received | TSE bulk CSV: `receitas_candidatos_{year}_{UF}.csv` | Donor name, CPF/CNPJ, amount, date, source type |
| Funding source breakdown | Field `DS_FONTE_RECEITA` | `FUNDO ESPECIAL` (FEFC), `FUNDO PARTIDARIO`, `OUTROS RECURSOS` |
| Donation origin | Field `DS_ORIGEM_RECEITA` | `Recursos de partido politico`, `Recursos de pessoas fisicas`, `Recursos proprios`, `Doacoes pela Internet`, etc. |
| Campaign spending | TSE bulk CSV: `despesas_candidatos_{year}_{UF}.csv` | Payee name, CNPJ, amount, date, category |
| Donation type | Field `DS_ESPECIE_RECEITA` | `Transferencia eletronica`, `Deposito em especie`, `Cartao de credito`, etc. |

**Data available for:** 2002, 2004, 2006, 2008, 2010, 2012, 2014, 2016, 2018, 2020, 2022, 2024. Federal elections in even years.

**Bulk download URL pattern:**
```
https://cdn.tse.jus.br/estatistica/sead/odsele/prestacao_contas/prestacao_de_contas_eleitorais_candidatos_{YEAR}.zip
```

#### Fundo Partidario (Permanent Party Fund)

| Data | Source | Fields |
|------|--------|--------|
| How much each party receives monthly | TSE Fundo Partidario page | Party, monthly amount (duodecimo), annual total |
| How parties spend it (annual) | TSE bulk: `prestacao_contas_anual_partidaria_{YEAR}.zip` | Party, expense description, amount, source (FP/FEFC/other) |
| Distribution criteria | TSE | 95% proportional to Camara votes, 5% equal |

**Bulk download URL:**
```
https://cdn.tse.jus.br/estatistica/sead/odsele/prestacao_contas_anual_partidaria/prestacao_contas_anual_partidaria_{YEAR}.zip
```

**Web consultation:** `https://divulgaspca.tse.jus.br/`

#### FEFC / Fundo Eleitoral (Campaign Fund)

| Data | Source | Fields |
|------|--------|--------|
| Allocation per party | TSE FEFC calculation spreadsheet | Party, total FEFC allocated |
| FEFC → candidate tracing | Campaign revenue files, `DS_FONTE_RECEITA = "FUNDO ESPECIAL"` | How much FEFC each candidate received |
| Gender/race distribution | TSE bulk: `fefc_fp_{YEAR}.zip` | FEFC by party, gender, race |

**Traceability chain:** FEFC total → Party allocation → Candidate receipt (via `DS_FONTE_RECEITA` + `DS_ORIGEM_RECEITA` fields in campaign revenue CSV)

**Bulk download URL:**
```
https://cdn.tse.jus.br/estatistica/sead/odsele/fefc_fp/fefc_fp_{YEAR}.zip
```

#### Emendas Parlamentares

| Data | Source | Fields |
|------|--------|--------|
| All emendas by author | Portal da Transparencia bulk download | Author, type, value (empenhado/liquidado/pago), municipality, function, subfuncao |
| Emendas de Relator (RP-9) | Same dataset, `Tipo de Emenda = "Emenda de Relator"` | The controversial "secret budget" |
| Emendas Pix (RP-8 TE) | Same dataset, `Tipo de Emenda = "Emenda Individual - Transferencias Especiais"` | Direct transfers with minimal oversight |
| Beneficiaries | `EmendasParlamentares_PorFavorecido.csv` | CNPJ/CPF of who actually received the money |

**Bulk download:**
```
https://dadosabertos-download.cgu.gov.br/PortalDaTransparencia/saida/emendas-parlamentares/EmendasParlamentares.zip
```

**API:** `https://api.portaldatransparencia.gov.br/api-de-dados/emendas` (requires free API key)

#### Asset Evolution

| Data | Source | Fields |
|------|--------|--------|
| Declared assets per election | TSE bulk: `bens_candidato_{year}.csv` | Asset type, description, declared value |
| Cross-election comparison | Match candidate across years by name + CPF (older datasets have full CPF) | Total patrimony 2018 vs 2022 vs 2026 |

**DivulgaCandContas API:**
```
GET /divulga/rest/v1/candidatura/buscar/{year}/{electionId}/{stateId}/{candidateId}/bens
```

#### Donor Cross-References

| Cross-Reference | Pipeline | Red Flag |
|-----------------|----------|----------|
| Donor CNPJ → CEIS/CNEP | Query Portal da Transparencia sanctions API with donor CNPJ | "Funded by sanctioned company" |
| Donor CNPJ → Government contracts | Query PNCP/ComprasGov with donor CNPJ | "Campaign donor received R$X in government contracts" |
| Donor CNPJ → Receita Federal QSA | Match donor CNPJ partners with other politicians | "Cross-funding network detected" |

---

### C. Performance

#### Voting Record

| Data | Source | Endpoint |
|------|--------|----------|
| All roll call votes | Camara API | `GET /votacoes/{id}/votos` |
| How a deputy voted | Camara API | `GET /deputados/{id}/votacoes` |
| Vote values | — | Sim, Nao, Abstencao, Obstrucao, Art. 17 |
| Senator votes | Senado API | `GET /senador/{codigo}/votacoes` |

#### Attendance

| Data | Source | Endpoint |
|------|--------|----------|
| Plenary sessions attended | Camara API | `GET /deputados/{id}/eventos` |
| Committee meetings | Camara API | Same endpoint, filter by event type |
| Attendance % | Computed | sessions_attended / total_sessions |
| Senator attendance | Senado API | `GET /senador/{codigo}/frequencia?ano={ano}` |

**Portal:** `https://www.camara.leg.br/deputados/{id}/presenca-em-plenario`

#### Bills Authored

| Data | Source | Endpoint |
|------|--------|----------|
| All bills by deputy | Camara API | `GET /deputados/{id}/proposicoes` |
| Bill details | Camara API | `GET /proposicoes/{id}` — type (PL, PEC, PLP), status, summary |
| Bill authors | Camara API | `GET /proposicoes/{id}/autores` |
| Bill progress | Camara API | `GET /proposicoes/{id}/tramitacoes` |
| Senator bills | Senado API | `GET /materia/pesquisa/lista?autor={nome}` |

#### CEAP/CEAPS Spending

| Data | Source | Endpoint |
|------|--------|----------|
| Deputy expenses | Camara API | `GET /deputados/{id}/despesas?ano={ano}&mes={mes}` |
| Fields | — | Category, date, amount, payee name, payee CNPJ, receipt URL |
| Senator expenses | Senado API | `GET /senador/{codigo}/despesas?ano={ano}` |
| Bulk download | Camara | `https://www.camara.leg.br/cota-parlamentar/` (CSV) |
| Bulk download | Senado | `https://www12.senado.leg.br/transparencia/dados-abertos-transparencia/dados-abertos-ceaps` |

#### Caucus Memberships

| Data | Source | Endpoint |
|------|--------|----------|
| Deputy's caucuses | Camara API | `GET /deputados/{id}/frentes` |
| All caucuses | Camara API | `GET /frentes` |
| Caucus members | Camara API | `GET /frentes/{id}/membros` |

---

### Ficha Limpa & Sanctions

#### TSE Ficha Limpa Status

| Data | Source | Fields |
|------|--------|--------|
| Registration status | TSE bulk: `candidatos_{year}.csv` | `DS_SITUACAO_CANDIDATURA`: DEFERIDO, INDEFERIDO, INDEFERIDO COM RECURSO, CASSADO, CANCELADO, RENUNCIA |
| Historical status | Query across all election years candidate participated in | Track status changes over time |
| Candidate details | DivulgaCandContas API | `GET /candidatura/buscar/{year}/{electionId}/{stateId}/{candidateId}` |

**Display logic:**
- `DEFERIDO` → Green badge "Ficha Limpa"
- `INDEFERIDO` → Red badge "Candidatura Indeferida"
- `INDEFERIDO COM RECURSO` → Yellow badge "Indeferida (em recurso)"
- `CASSADO` → Red badge "Mandato Cassado"
- Historical changes flagged: "Was DEFERIDO in 2018, INDEFERIDO in 2022"

#### Sanctions Cross-Reference

| Cross-Reference | Pipeline |
|-----------------|----------|
| Politician CPF → Receita Federal QSA → Company CNPJ → CEIS match | "Partner in sanctioned company" |
| Politician CPF → Receita Federal QSA → Company CNPJ → CNEP match | "Partner in punished company (Lei Anticorrupcao)" |
| Politician CPF → Receita Federal QSA → Company CNPJ → CEPIM match | "Partner in barred NGO" |
| Politician CPF → TCU irregular accounts | "Accounts judged irregular by TCU" |

**Data sources:**
- CEIS/CNEP/CEPIM: `https://api.portaldatransparencia.gov.br/` (free API key)
- Receita Federal QSA: `https://dadosabertos.rfb.gov.br/CNPJ/` (bulk download)
- TCU: `https://portal.tcu.gov.br/responsabilizacao-publica/contas-julgadas-irregulares/`

---

### History / Meta

| Data | Source |
|------|--------|
| Party switches | Compare TSE filiacao data + candidate registrations across elections |
| Photo | TSE DivulgaCandContas or Camara/Senado API |
| Education, birth date, occupation | TSE candidate data |
| Electoral district | TSE candidate data |

---

## Features

### 1. Ranking / Leaderboard

Sortable table of all 594 legislators. Sortable columns:

| Column | Data |
|--------|------|
| Name | — |
| Party | — |
| State | — |
| Attendance % | Computed from Camara/Senado API |
| Bills approved | Count from proposicoes |
| CEAP total spent | Sum from expenses API |
| Emendas total R$ | Sum from Portal da Transparencia |
| Asset growth % | (2022 assets - 2018 assets) / 2018 assets |
| FEFC received R$ | From TSE campaign finance |
| Private donations R$ | From TSE campaign finance |
| Ficha Limpa | Badge |
| Sanctions | Count |

Filters: party, state, position (deputy/senator), caucus.

### 2. Side-by-Side Comparison

Select 2-3 politicians → comparison table with all metrics. Shareable URL: `/compare?ids=123,456,789`

### 3. Search

Full-text search by politician name. Autocomplete from the 594 profiles.

### 4. Shareable City Cards

Auto-generated social media cards (Open Graph) per politician with key stats. Designed for sharing on Twitter/WhatsApp.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Framework | Next.js (App Router) |
| Language | TypeScript |
| Styling | Tailwind CSS |
| Database | Supabase (PostgreSQL) |
| Data Pipeline | Python (pandas) + GitHub Actions (nightly) |
| Upstream Data | TSE bulk CSVs + Camara/Senado APIs + Portal da Transparencia API + BigQuery (basedosdados.org) |
| Deploy | Vercel (free tier) |
| Cost | $0 infrastructure |

### Database Schema (Supabase)

**Core tables:**

```
politicians
├── id (PK)
├── tse_candidate_id
├── camara_id / senado_id
├── full_name
├── ballot_name
├── cpf_masked
├── party
├── state
├── position (deputy/senator)
├── photo_url
├── birth_date
├── education
├── occupation
├── ficha_limpa_status
├── ficha_limpa_history (JSONB)
├── attendance_pct
├── bills_authored
├── bills_approved
├── ceap_total
├── emendas_total
├── asset_current
├── asset_previous
├── asset_growth_pct
├── fefc_received
├── private_donations_total
├── sanctions_count
├── updated_at

campaign_donations
├── id (PK)
├── politician_id (FK)
├── election_year
├── donor_name
├── donor_cpf_cnpj
├── amount
├── date
├── source_type (FEFC / Fundo Partidario / Private / Self)
├── origin_type
├── is_sanctioned_donor (boolean)
├── donor_has_gov_contracts (boolean)

emendas
├── id (PK)
├── politician_id (FK)
├── code
├── year
├── type (Individual / Relator / Pix / Bancada / Comissao)
├── value_committed
├── value_paid
├── municipality_ibge
├── municipality_name
├── state
├── function
├── subfunction

ceap_expenses
├── id (PK)
├── politician_id (FK)
├── year
├── month
├── category
├── amount
├── payee_name
├── payee_cnpj
├── receipt_url

votes
├── id (PK)
├── politician_id (FK)
├── vote_session_id
├── date
├── subject
├── vote (sim/nao/abstencao/obstrucao/art17)

bills
├── id (PK)
├── politician_id (FK)
├── bill_type (PL/PEC/PLP/PDL)
├── bill_number
├── year
├── summary
├── status (tramitando/aprovado/arquivado)

assets
├── id (PK)
├── politician_id (FK)
├── election_year
├── asset_type
├── description
├── declared_value

sanctions_crossref
├── id (PK)
├── politician_id (FK)
├── type (CEIS/CNEP/CEPIM/TCU)
├── company_cnpj
├── company_name
├── sanction_detail
├── link_type (partner/donor)

caucuses
├── id (PK)
├── politician_id (FK)
├── caucus_name
├── caucus_id
```

### Data Pipeline

```
NIGHTLY (GitHub Actions)
│
├── Camara API → fetch all deputies, expenses, votes, bills, attendance, caucuses
├── Senado API → fetch all senators, expenses, votes, attendance
├── Portal da Transparencia API → fetch emendas by author
├── Compute: attendance %, asset growth %, rankings
├── Cross-reference: donor CNPJs → CEIS/CNEP → flag sanctioned donors
├── Cross-reference: donor CNPJs → PNCP contracts → flag donors with contracts
├── Upsert all data into Supabase
│
PERIODIC (manual or quarterly)
│
├── TSE bulk downloads → campaign finance, candidate data, assets, filiacao
├── Receita Federal CNPJ bulk → QSA (company partners)
├── TCU irregular accounts
└── Recompute sanctions cross-references
```

---

## Key Design Decisions

### 1. TSE Ficha Limpa as Criminal Record Proxy

**Decision:** Use TSE candidate registration status (DEFERIDO/INDEFERIDO/CASSADO) as the proxy for criminal records, instead of querying court systems directly.

**Why:** There is no free public API that accepts a CPF and returns court cases. DataJud (CNJ) searches by name only, producing false positives. Court certificate systems have CAPTCHAs and return binary PDFs, not structured data.

**What Ficha Limpa covers:** Convictions by collegiate courts for corruption, money laundering, electoral fraud, crimes against public administration; impeachment; TCU account rejections; removal from public office; electoral abuse convictions.

**What it misses:** Ongoing investigations, single-judge convictions under appeal, cases involving family/associates.

**V2 option (documented, not in MVP):** Escavador API (`api.escavador.com`) accepts CPF and returns all court cases for R$0.10-0.20/query. Total cost for all 594 politicians: ~R$60-200. Python SDK available. See Escavador Research section.

### 2. No Fuzzy Name Matching for Criminal Records

**Decision:** Refuse to publish criminal case associations based on fuzzy name matching.

**Why:** Publishing "this politician has criminal cases" based on automated name matching carries defamation risk. "Jose Carlos de Souza Lima" the deputy vs "Jose Carlos de Souza Lima" the unrelated citizen — getting this wrong destroys credibility and invites lawsuits.

### 3. Deputies + Senators from Day One

**Decision:** Cover all 594 federal legislators (513 deputies + 81 senators) at launch.

**Why:** October 2026 elections include both chambers. Voters need both. The Senado API mirrors the Camara API closely enough that the marginal effort is small.

### 4. Rankings as Landing Page

**Decision:** Landing page is a sortable ranking table, not a search box.

**Why:** Rankings generate headlines and social sharing ("The 10 deputies who missed the most votes"). Search is discoverable but secondary. The ranking IS the product for most users; profiles are the depth for journalists.

### 5. Shared Data Layer with Project A

**Decision:** Same Supabase instance as the City Transparency Map. Shared tables for emendas, CNPJ data, sanctions.

**Why:** The emendas data connects both products. A politician's emendas appear in their profile (Project B) and in the city's money-in layer (Project A). Single source of truth, no duplication.

---

## V2: Escavador Integration (Criminal Records)

### Research Summary

**Escavador API** (`api.escavador.com`) is the only commercial service that accepts a CPF and returns all court cases programmatically.

| Attribute | Detail |
|-----------|--------|
| Auth | Bearer token (OAuth), register at dashboard |
| Endpoint | `GET /api/v2/envolvido/processos` (accepts CPF) |
| Returns | All court cases: case number, parties, dates, source tribunal, status |
| Coverage | 90+ Brazilian courts (all TJs, TRFs, STF, STJ, TSE, TRTs) |
| Pricing | R$0.10-0.20 per query. 24h dedup (multiple services same process = max price only) |
| Cost estimate | ~R$60-200 for all 594 politicians |
| Rate limit | 500 req/min |
| SDK | Python: `pip install escavador` → `Processo.por_cpf(cpf="...")` |
| Limitation | Returns ALL case types (civil, labor, criminal). Must filter criminal client-side. |
| Limitation | Sealed cases (segredo de justica) excluded |

**Alternatives:**
- JusBrasil API: Commercial, no public pricing, requires sales contract
- JUDIT: 90+ courts, MCP integration, no public pricing
- Codilo: CAPTCHA bypass for courts, no public pricing
- Infosimples: R$0.08-0.20/query, volume discounts
- Predictus: CPF/CNPJ lookup, no public pricing

**Implementation plan for v2:**
1. Register at Escavador, get API token
2. Load all 594 politician CPFs (from TSE historical data 2002-2012 for full CPFs, or from Camara/Senado data)
3. Query Escavador for each CPF
4. Filter results: keep only criminal case classes (Acao Penal, Inquerito, AIJE, AIME)
5. Store in `criminal_cases` table in Supabase
6. Display in Ficha Limpa tab with source links to original court
7. Refresh quarterly

---

## MVP Scope

### In MVP

- Ranking page with all 594 legislators, sortable by any metric
- Politician profile with Money Trail, Performance, Ficha Limpa tabs
- Campaign funding breakdown (FEFC vs Fundo Partidario vs private)
- Top donors list with CNPJ
- Emendas authored (with type flags for RP-9/RP-8)
- Asset evolution (2018 vs 2022)
- Voting record (all roll calls)
- Attendance %
- Bills authored (count + status)
- CEAP spending (total + top suppliers)
- Caucus memberships
- Ficha Limpa status (DEFERIDO/INDEFERIDO/CASSADO, historical)
- CEIS/CNEP sanctions cross-reference
- Side-by-side comparison (2-3 politicians)
- Search by name
- Filter by party, state, position
- Shareable politician cards (OG meta tags)
- Mobile-friendly responsive design

### Post-MVP

- Escavador integration (criminal cases via CPF lookup)
- Donor → government contract cross-reference
- Donor network visualization (who funds whom)
- Party spending analysis (Fundo Partidario annual accounts)
- 2026 candidate profiles (when registrations open July 2026)
- Push notifications for new data (voting record updates, new emendas)
- API for journalists/researchers
- Integration with Project A (link to city profiles from emenda destinations)
- Querido Diario integration (gazette mentions of politician/donors)

---

## Timeline

| Phase | Scope | Target |
|-------|-------|--------|
| Data pipeline | TSE bulk ingestion + Camara/Senado API integration | Week 1-2 |
| Core profiles | Politician profiles with all MVP data | Week 3-4 |
| Rankings + comparison | Sortable leaderboard, side-by-side compare | Week 5 |
| Cross-references | Donor → sanctions, asset growth flags | Week 6 |
| Polish + launch | Mobile, OG cards, shareable URLs | Week 7-8 |
| 2026 candidates | Integrate new registrations as they open | July 2026 |
| Escavador | Criminal records via CPF | Post-launch |
