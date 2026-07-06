# Salutar Saúde — Análise de Dados

Case técnico de Analista de Dados: uma plataforma de dados completa para uma corretora de saúde fictícia (**Salutar Saúde**), do dado cru à recomendação de negócio.

Stack: **Databricks** (Unity Catalog, Delta Lake), **PySpark/SQL**, arquitetura **medallion** (bronze → silver → gold) e um modelo dimensional **star schema** (Kimball). Catálogo/schema: `homologacao.salutar_saude`.

> **Data de referência do case:** `hoje = 30/06/2025` (âncora de todas as janelas temporais).

---

## Estrutura do repositório

| Pasta | Conteúdo |
|---|---|
| [`silver/`](silver/) | Notebooks de transformação **RAW → Silver**: limpeza, tipagem, padronização de categorias, normalização de datas/moeda BR e deduplicação, um por fonte de dados. |
| [`gold/`](gold/) | Notebook de construção da camada **Gold** (star schema: 6 dimensões + 3 fatos) e a documentação técnica do modelo (grãos, chaves, linhagem, guia de JOINs). |
| [`analises/`](analises/) | Análises de negócio: **comercial** (planos, empresas, corretores, contratos a expirar, sinistralidade) e **inteligência de sinistros** (Pareto, segmentação, padrões de utilização, recomendações). |
| [`dq/`](dq/) | Suíte de **qualidade de dados** (74 testes automáticos sobre a Gold) e o guia de qualidade em linguagem de negócio. |
| [`insights/`](insights/) | **Exploração avançada e independente**: sazonalidade, seleção adversa, funil PS→cirurgia, concentração de receita (HHI), eficiência de rede e variabilidade de preços. |
| [`docs/`](docs/) | Apresentação executiva do case (`.pptx`). |

---

## Camada Silver — `silver/`

Cada notebook lê uma tabela `raw_*` e produz a `silver_*` correspondente via MERGE incremental (Delta Lake), aplicando:

- Padronização de categorias despadronizadas (ex.: `Ativo` / `ativo` / `ATIVO` / `"Ativo "` → valor canônico).
- Normalização de UF, sexo, datas (`DD/MM/YYYY` → `YYYY-MM-DD`) e moeda BR.
- Deduplicação defensiva de chaves (ex.: `empresa_id`, `contrato_id`).
- Data Quality inline (contagem de nulos, valores inesperados, correspondência com a RAW).

| Notebook | Fonte | Grão |
|---|---|---|
| `silver_operadoras` | `raw_operadoras` | 1 linha / operadora |
| `silver_empresas` | `raw_empresas` | 1 linha / empresa |
| `silver_planos` | `raw_planos` | 1 linha / plano |
| `silver_corretores` | `raw_corretores` | 1 linha / corretor |
| `silver_beneficiarios` | `raw_beneficiarios` | 1 linha / beneficiário |
| `silver_contratos` | `raw_contratos` | 1 linha / contrato |
| `silver_utilizacao` | `raw_utilizacao` | 1 linha / evento (sinistro) |

## Camada Gold — `gold/`

Star schema (Kimball) exposto em `gold_*`:

- **Dimensões (6):** `dim_operadora`, `dim_empresa`, `dim_plano`, `dim_corretor`, `dim_data` (conformada, 2018–2030), `dim_beneficiario`.
- **Fatos (3):** `fat_contratos` (grão: contrato), `fat_utilizacao` (grão: evento), `fat_sinistralidade` (grão: empresa × mês, pré-agregado).

A **sinistralidade** (`custo de sinistros ÷ receita de prêmios`) é materializada em `fat_sinistralidade` com premissas de cálculo documentadas (receita de prêmios ativos, custo por data do evento, margem técnica, tratamento de receita zero). Detalhes em [`gold/gold_star_schema_docs.md`](gold/gold_star_schema_docs.md).

## Qualidade de Dados — `dq/`

74 testes automáticos em 6 categorias (Domínio, Regras de Negócio, Integridade, Unicidade, Volume, Completude) sobre as 9 tabelas Gold. Um `FAIL` bloqueia o pipeline. Guia em linguagem de negócio em [`dq/gold_dq_guia_negocio.md`](dq/gold_dq_guia_negocio.md).

## Análises & Insights — `analises/` e `insights/`

Respondem, com evidência, onde o time deve atuar: distribuição de custo (high-cost claimants), segmentação de coortes, padrões de utilização e uso evitável, empresas em prejuízo técnico, além de achados avançados (seleção adversa, sazonalidade, funil de escalação, concentração de receita e variabilidade de preços).

---

## Como reproduzir

1. Importe os notebooks para um workspace **Databricks** com Unity Catalog habilitado.
2. Garanta que a camada `raw_*` exista em `homologacao.salutar_saude`.
3. Execute na ordem: `silver/` → `gold/` → `dq/` → `analises/` e `insights/`.
4. Os notebooks usam `spark`/`%sql`; ajuste `CATALOG` e `SCHEMA` no topo de cada um se necessário.

## Principais suposições

- `hoje = 30/06/2025` como âncora temporal.
- RAW é imutável; todo tratamento ocorre na transformação, com regras documentadas (não adivinhadas).
- Sinistralidade **não** inclui projeção de IBNR (premissa explícita).
- Limiar de risco de sinistralidade: 🔴 prejuízo técnico `> 100%` · 🟡 atenção `> 80%`.
