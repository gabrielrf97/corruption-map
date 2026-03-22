# Data Sources Reference

Complete reference of all freely available data sources for the City Transparency Map.

---

## Quick Reference: API Endpoints

### No Authentication Required

| Source | Base URL | Data |
|--------|----------|------|
| SICONFI/FINBRA | `https://apidatalake.tesouro.gov.br/ords/siconfi/tt/` | Municipal budget execution |
| IBGE Malhas v3 | `https://servicodados.ibge.gov.br/api/v3/malhas/` | Municipality GeoJSON/TopoJSON |
| IBGE Localidades | `https://servicodados.ibge.gov.br/api/v1/localidades/` | IBGE codes, names |
| IBGE SIDRA | `https://apisidra.ibge.gov.br/` | Population, GDP |
| Camara dos Deputados | `https://dadosabertos.camara.leg.br/api/v2/` | Deputies, expenses, votes |
| Senado Federal | `https://legis.senado.leg.br/dadosabertos/` | Senators, expenses |
| PNCP | `https://pncp.gov.br/api/pncp/v1/` | Municipal/federal procurement |
| SIOPS | `https://siops-consulta-publica-api.saude.gov.br` | Municipal health spending |
| CNES/DataSUS | `https://apidadosabertos.saude.gov.br/` | Health facilities |
| INEP | `http://api.dadosabertosinep.org/v1` | IDEB scores |
| Querido Diario | `https://queridodiario.ok.org.br/api/` | Municipal gazettes |
| TSE DivulgaCandContas | `https://divulgacandcontas.tse.jus.br/divulga/rest/v1/` | Candidates, donations |
| TSE Dados Abertos | `https://dadosabertos.tse.jus.br/` | Election bulk data (CKAN) |
| TCE-SP | `https://transparencia.tce.sp.gov.br/api/` | SP municipal expenses |
| TCE-RJ | `https://dados.tcerj.tc.br/api/v1/` | RJ contracts, bids, payroll |
| TCE-PE | `https://sistemas.tce.pe.gov.br/DadosAbertos/` | PE expenses, contracts |
| TCE-RS | `https://dados.tce.rs.gov.br/` | RS municipal data (CKAN) |
| TCE-SC | `https://servicos.tcesc.tc.br/endpoints-portal-transparencia/` | SC municipal accounts |
| TransfereGov | `https://api.transferegov.sistema.gov.br/` | Federal agreements |
| ComprasGov | `https://compras.dados.gov.br/v1/` | Federal procurement |
| CGU EBT | `https://mbt.cgu.gov.br/` | Transparency scores |
| SNIS | `https://app4.mdr.gov.br/serieHistorica/` | Sanitation data |

### Free Registration Required

| Source | Base URL | Registration |
|--------|----------|-------------|
| Portal da Transparencia | `https://api.portaldatransparencia.gov.br/` | Free API key via portal |
| Brasil.IO | `https://api.brasil.io/v1/` | Free token via registration |
| basedosdados.org | BigQuery SQL | Google Cloud account (1TB/month free) |

### Bulk Downloads (No Auth)

| Source | URL | Format | Size |
|--------|-----|--------|------|
| Receita Federal CNPJ | `https://dadosabertos.rfb.gov.br/CNPJ/` | CSV (zipped) | ~25GB uncompressed |
| TSE Elections | `https://cdn.tse.jus.br/estatistica/sead/odsele/` | CSV (zipped) | ~500MB-2GB per cycle |
| Portal da Transparencia | `https://portaldatransparencia.gov.br/download-de-dados` | CSV (zipped) | 2-10GB per dataset |

---

## Detailed Source Documentation

### 1. SICONFI — Municipal Budget Execution

The single most important source for municipal fiscal data. Covers 5,496 of 5,570 municipalities (98.6%).

**Base URL:** `https://apidatalake.tesouro.gov.br/ords/siconfi/tt/`

**Endpoints:**

| Endpoint | Description | Update Frequency |
|----------|-------------|-----------------|
| `/entes` | List all entities with IBGE code, population, CNPJ | Static |
| `/dca` | Annual balance sheets | Annual |
| `/rreo` | Bimonthly budget execution | Bimonthly |
| `/rgf` | Fiscal management report | Quadrimesterly |
| `/msc_orcamentaria` | Accounting balance matrix | Monthly |

**Key parameters:**
- `an_exercicio` — fiscal year (e.g., 2023)
- `no_anexo` — annex name (e.g., "DCA-Anexo+I-D")
- `id_ente` — IBGE code of municipality
- `nr_periodo` — period (bimester 1-6 for RREO, quadrimester for RGF)

**Data available:** Budget revenues, budget expenses, expenses by function (health, education, personnel, security, urbanismo, saneamento), balance sheet. Historical data from 1989.

**Also on BigQuery:** `basedosdados.br_me_siconfi.municipio_despesas_funcao` (1996-2023)

---

### 2. Portal da Transparencia API

**Base URL:** `https://api.portaldatransparencia.gov.br/api-de-dados/`
**Docs:** `http://portaldatransparencia.gov.br/api-docs`
**Auth:** Header `chave-api-dados: <your-key>` (free registration)
**Rate limit:** ~100 req/min

**Key endpoints:**

| Endpoint | Fields |
|----------|--------|
| `/transferencias` | mesAno, codigoIBGE, valor, tipoTransferencia, orgaoSuperior |
| `/convenios` | numero, valorGlobal, valorLiberado, orgaoConcedente, convenente, municipio |
| `/emendas` | codigoEmenda, nomeAutor, tipoEmenda, localidadeDoGasto, valorPago, funcao |
| `/despesas/documentos` | codigoFavorecido, nomeFavorecido, cpfCnpjFavorecido, codigoMunicipioIBGE |
| `/ceis` | razaoSocial, cnpjCpf, tipoSancao, orgaoSancionador, dataInicio |
| `/cnep` | same as CEIS (punished companies) |
| `/cepim` | barred NGOs |
| `/licitacoes` | numero, modalidade, objeto, valorEstimado, municipio |

---

### 3. IBGE Geographic Data

**Municipality boundaries:**
```
https://servicodados.ibge.gov.br/api/v3/malhas/paises/BR?formato=application/vnd.geo+json&qualidade=intermediaria&intrarregiao=municipio
```
- Formats: GeoJSON or TopoJSON
- Quality: `minima`, `intermediaria`, `maxima`
- Per state: `/malhas/estados/{UF_code}?...&intrarregiao=municipio`

**Municipality list:**
```
https://servicodados.ibge.gov.br/api/v1/localidades/municipios
```
- Returns: id (7-digit IBGE code), nome, UF, region

**Population (Censo 2022):**
```
https://apisidra.ibge.gov.br/values/t/4714/n6/all/v/93/p/last
```

**GDP per municipality:**
```
https://apisidra.ibge.gov.br/values/t/5938/n6/all/v/37/p/last
```

**Municipality centroids:** `https://github.com/kelvins/municipios-brasileiros` (CSV with lat/lon)

---

### 4. SIOPE — Education Spending

**URL:** `https://www.fnde.gov.br/siope/`
**Municipal reports:** `https://www.fnde.gov.br/siope/relatoriosMunicipais.jsp`
**Open data:** `https://dados.gov.br/dados/conjuntos-dados/sistema-de-informacoes-sobre-orcamentos-publicos-em-educacao-siope`

**Data per municipality:**
- How much invested in education
- Compliance with 25% constitutional minimum
- Revenue and expenditure breakdown for education
- FUNDEB application data
- Update: Annual (deadline April 30)

---

### 5. SIOPS — Health Spending

**API (Swagger):** `https://siops-consulta-publica-api.saude.gov.br`
**Open Data:** `https://dadosabertos.saude.gov.br/dataset/siops`

**Data per municipality:**
- Health expenditure by source and sub-function
- Compliance with 15% minimum (LC 141/2012)
- Update: Bimonthly

---

### 6. CNES/DataSUS — Health Infrastructure

**API:** `https://apidadosabertos.saude.gov.br/`
**ElastiCNES:** `https://elasticnes.saude.gov.br/`
**Downloads:** `https://cnes.datasus.gov.br/pages/downloads/arquivosBaseDados.jsp`

**Data per municipality:**
- All health establishments (hospitals, clinics, UBS)
- Number of beds (general + complementary) by specialty
- Health professionals (doctors, nurses)
- SUS linkage status
- Update: Monthly

---

### 7. INEP / IDEB — Education Quality

**API:** `http://api.dadosabertosinep.org/v1`

**Key endpoints:**
- `/ideb/municipio/{codigo_municipio}.json` — IDEB by municipality
- `/ideb/escolas.json?codigo_municipio=XXXX` — All schools in municipality

**Data:** IDEB scores (anos iniciais + finais), school count, teacher count, enrollment
**Update:** Biennial (IDEB), annual (Censo Escolar)

---

### 8. SNIS — Sanitation

**Historical series:** `https://app4.mdr.gov.br/serieHistorica/`
**Open data:** `https://dados.gov.br/dados/conjuntos-dados/snis---srie-histrica`
**BigQuery:** `basedosdados.br_mdr_snis`

**Data per municipality:**
- Water coverage %
- Sewage coverage %
- Water losses
- Tariffs
- Operational/quality indicators
- Update: Annual

---

### 9. PNCP — Municipal Procurement

**API:** `https://pncp.gov.br/api/pncp/v1/`
**Swagger:** `https://pncp.gov.br/api/pncp/swagger-ui/index.html`

**Key endpoints:**
- `GET /orgaos/{cnpj}/compras` — procurement by entity
- `GET /orgaos/{cnpj}/compras/{ano}/{seq}/contratos` — contracts with supplier CNPJ
- `GET /contratacoes/publicacao` — search by date range

**Coverage:** Federal + state + municipal entities (under Lei 14.133/2021). Growing but incomplete for small municipalities (6-year transition period).

---

### 10. TCE APIs — State Audit Courts

#### TCE-SP (Sao Paulo) — 645 municipalities
```
https://transparencia.tce.sp.gov.br/api/{formato}/despesas/{municipio}/{exercicio}/{mes}
https://transparencia.tce.sp.gov.br/api/{formato}/receitas/{municipio}/{exercicio}/{mes}
https://transparencia.tce.sp.gov.br/api/{formato}/municipios
```
Formats: `json` or `xml`. Fields: organ, month, supplier ID/name, expense amount.

#### TCE-RJ (Rio de Janeiro) — 92 municipalities
```
https://dados.tcerj.tc.br/api/v1/
```
15+ endpoints: `compras_diretas_municipio`, `contratos_municipio`, `licitacoes`, `receitas_municipio`, `empenho_municipio`, `licitante_vencedor_municipio`, `licitante_perdedor_municipio`, `obras_paralisadas`, `compras_covid_municipio`, etc.
Pagination: `inicio` (offset), `limite` (default 1000).

#### TCE-PE (Pernambuco) — 185 municipalities
```
https://sistemas.tce.pe.gov.br/DadosAbertos/
```
Datasets: Receitas, Despesas, Fornecedores, Licitacoes, Contratos, Obras, Pessoal.

#### TCE-RS (Rio Grande do Sul) — 497 municipalities
```
https://dados.tce.rs.gov.br/
```
CKAN-based. 69,300+ datasets. Empenhos, education/health spending indices, contracts.

#### TCE-SC (Santa Catarina) — 295 municipalities
```
https://servicos.tcesc.tc.br/endpoints-portal-transparencia/
```
e-SFINGE system. JSON/XML.

#### TCE-MG (Minas Gerais) — 853 municipalities
```
https://dadosabertos.tce.mg.gov.br/
```
SICOM system. All 853 municipalities submit accounting data.

#### TCE-PB (Paraiba) — 223 municipalities
```
https://dados-abertos.tce.pb.gov.br/
```
SAGRES system. Pipe-delimited text files, GZip compressed.

#### Other TCEs (less structured data):
- TCE-BA: `https://www.tce.ba.gov.br/dados-abertos`
- TCE-CE: `https://etransparencia.tce.ce.gov.br/`
- TCE-PR: `https://www.tce.pr.gov.br/transparencia-do-tce-pr/dados-abertos-tce-pr.htm`
- TCE-ES: `https://paineldecontrole.tcees.tc.br/dadosAbertos`

---

### 11. Additional Municipal Data

| Source | Data | URL | Update |
|--------|------|-----|--------|
| RAIS/CAGED | Formal employment per municipality | BigQuery: `basedosdados.br_me_rais` | Annual/Monthly |
| MUNIC (IBGE) | Municipal governance survey | SIDRA: `sidra.ibge.gov.br/pesquisa/munic/tabelas` | Annual |
| CGU EBT | Transparency compliance score | `mbt.cgu.gov.br` | Periodic |
| CadUnico/Bolsa Familia | Families in poverty, benefit amounts | Portal da Transparencia API | Monthly |
| Receita Federal arrecadacao | Tax collection per municipality | `gov.br/receitafederal/.../receitadata/arrecadacao` | Monthly |
| FNS | Health transfers | `consultafns.saude.gov.br` | Daily |
| FNDE | Education transfers (FUNDEB, PNAE, PDDE) | `dados.gov.br` CKAN | Periodic |
| Atlas Brasil | HDI per municipality | `atlasbrasil.org.br` | Census years |
| Querido Diario | Municipal gazette full text | `queridodiario.ok.org.br/api/` | Daily |

---

## Universal Join Key

All datasets are linked by the **7-digit IBGE municipality code** (`cod_ibge` / `codigo_ibge`).

Examples:
- Sao Paulo: `3550308`
- Rio de Janeiro: `3304557`
- Maraba: `1504208`

This code appears in: SICONFI, Portal da Transparencia, TSE, CNPJ (Receita Federal), CNES, INEP, SNIS, RAIS, MUNIC, IBGE Malhas (as `codarea`), and virtually every government dataset.

---

## basedosdados.org — Unified BigQuery Access

Many of the above sources are pre-normalized and queryable via SQL on BigQuery:

| BigQuery Table | Original Source |
|----------------|----------------|
| `basedosdados.br_me_siconfi.municipio_despesas_funcao` | SICONFI |
| `basedosdados.br_me_siconfi.municipio_receitas_orcamentarias` | SICONFI |
| `basedosdados.br_cgu_portaldatransparencia.emendas_parlamentares` | Portal da Transparencia |
| `basedosdados.br_cgu_portaldatransparencia.transferencias` | Portal da Transparencia |
| `basedosdados.br_cgu_portaldatransparencia.ceis` | CGU CEIS |
| `basedosdados.br_ms_cnes.*` | CNES/DataSUS |
| `basedosdados.br_inep_censo_escolar.*` | INEP |
| `basedosdados.br_inep_ideb.*` | INEP |
| `basedosdados.br_me_rais.*` | RAIS |
| `basedosdados.br_me_caged.*` | CAGED |
| `basedosdados.br_mdr_snis.*` | SNIS |
| `basedosdados.br_ibge_munic.*` | MUNIC |
| `basedosdados.br_ibge_pib.municipio` | IBGE PIB |
| `basedosdados.br_bd_diretorios_brasil.municipio` | Municipality directory (IBGE code, name, UF, lat, lon, population) |

**Access:** Free Google Cloud account, 1TB/month query processing free.
**Python:** `pip install basedosdados` → `basedosdados.read_sql(query, billing_project_id="your-project")`
