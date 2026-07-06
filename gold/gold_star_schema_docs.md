# Documentação Técnica — Gold Layer Salutar Saúde

| | |
|---|---|
| **Catálogo / Schema** | `homologacao.salutar_saude` |
| **Notebook de geração** | `gold_star_schema` (id: 1745300139045536) |
| **Gerado em** | 2026-07-04 |
| **Camada** | Gold (consumo analítico) |
| **Padrão** | Star Schema (Kimball) |

---

## Sumário

1. [Visão Geral](#visão-geral)
2. [Diagrama do Modelo](#diagrama-do-modelo)
3. [Dimensões](#dimensões)
   - [dim_operadora](#dim_operadora)
   - [dim_empresa](#dim_empresa)
   - [dim_plano](#dim_plano)
   - [dim_corretor](#dim_corretor)
   - [dim_data](#dim_data)
   - [dim_beneficiario](#dim_beneficiario)
4. [Fatos](#fatos)
   - [fat_contratos](#fat_contratos)
   - [fat_utilizacao](#fat_utilizacao)
   - [fat_sinistralidade](#fat_sinistralidade)
5. [Premissas de Cálculo — fat_sinistralidade](#premissas-de-cálculo--fat_sinistralidade)
6. [Linhagem de Dados](#linhagem-de-dados)
7. [Guia de JOINs](#guia-de-joins)
8. [Exemplos de Queries](#exemplos-de-queries)

---

## Visão Geral

A camada **Gold** expõe um modelo dimensional (Star Schema) sobre os dados de planos de saúde da Salutar Saúde. Todas as tabelas residem em `homologacao.salutar_saude` com prefixo `gold_`.

O modelo é composto por:

| Tipo | Tabelas | Total |
|---|---|---|
| Dimensões | `dim_operadora`, `dim_empresa`, `dim_plano`, `dim_corretor`, `dim_data`, `dim_beneficiario` | 6 |
| Fatos | `fat_contratos`, `fat_utilizacao`, `fat_sinistralidade` | 3 |

**Volumes reais (última execução):**

| Tabela | Registros |
|---|---:|
| `gold_dim_operadora` | 8 |
| `gold_dim_empresa` | 41 |
| `gold_dim_plano` | 28 |
| `gold_dim_corretor` | 25 |
| `gold_dim_data` | 4.748 |
| `gold_dim_beneficiario` | 1.558 |
| `gold_fat_contratos` | 109 |
| `gold_fat_utilizacao` | 6.480 |
| `gold_fat_sinistralidade` | 985 |

---

## Diagrama do Modelo

```
           ┌──────────────────┐
           │  dim_operadora   │
           │  PK: operadora_id│◄──────────────────────┐
           └──────────────────┘                       │
                                                      │
┌─────────────────┐  ┌──────────────────┐             │  ┌──────────────────┐
│  dim_empresa    │  │    dim_plano      │─────────────┘  │  dim_corretor    │
│  PK: empresa_id │  │  PK: plano_id    │                │  PK: corretor_id │
└────────┬────────┘  └────────┬─────────┘                └────────┬─────────┘
         │                   │                                    │
         │  ┌────────────────▼────────────────────────────────────▼──────────┐
         ├─►│                    fat_contratos                                │◄── dim_data
         │  │  PK: contrato_id                                                │    (vigências)
         │  │  FK: empresa_id · plano_id · corretor_id                        │
         │  │  Méd: num_vidas · valor_mensal · receita_total_mensal           │
         │  └────────────────┬────────────────────────────────────────────────┘
         │                   │
         │  ┌────────────────▼──────────────────┐  ┌──────────────────────┐
         │  │        dim_beneficiario            │  │       dim_data       │
         │  │  PK: beneficiario_id               │  │  PK: data (DATE)     │
         │  │  FK: contrato_id · empresa_id      │  │      data_id (INT)   │
         │  └──────────────────┬─────────────────┘  └──────────┬───────────┘
         │                     ▲                               │
         │    ┌────────────────┴───────────────────────────────▼────────────┐
         ├───►│                    fat_utilizacao                            │
         │    │  PK: evento_id                                               │
         │    │  FK: beneficiario_id · contrato_id · empresa_id · plano_id  │
         │    │  Méd: valor_sinistro                                         │
         │    └──────────────────────────────────────────────────────────────┘
         │
         │  ┌───────────────────────────────────────────────────────────────┐
         └─►│                   fat_sinistralidade                          │◄── dim_data
            │  PK: empresa_id × ano_mes                                     │    (via ano_mes)
            │  FK: empresa_id → dim_empresa · ano_mes → dim_data            │
            │  Méd: sinistralidade_pct · receita_premios · custo_sinistros  │
            │       margem_tecnica · custo_per_capita                       │
            └───────────────────────────────────────────────────────────────┘
```

---

## Dimensões

### dim_operadora

> **Grão:** 1 linha por operadora de saúde  
> **Fonte:** `silver_operadoras`  
> **Registros:** ~8  

| Coluna | Tipo | Nulável | PK/FK | Descrição |
|---|---|---|---|---|
| `operadora_id` | INT | NÃO | PK | Chave primária da operadora |
| `operadora_nome` | STRING | SIM | — | Nome comercial da operadora |

**Valores conhecidos de `operadora_nome`:**
Vitamed, Boavida Saúde, SulVida Seguros, NovaClin Intermédica, Sanare Saúde, Aurora Saúde, Unicare Coop, PlenaCare.

---

### dim_empresa

> **Grão:** 1 linha por empresa contratante  
> **Fonte:** `silver_empresas`  
> **Registros:** ~41  

| Coluna | Tipo | Nulável | PK/FK | Descrição |
|---|---|---|---|---|
| `empresa_id` | INT | NÃO | PK | Chave primária da empresa |
| `empresa_nome` | STRING | SIM | — | Razão social da empresa |
| `setor` | STRING | SIM | — | Setor econômico (ex.: Saúde, Tecnologia, Varejo, Indústria) |
| `porte` | STRING | SIM | — | `Pequena` \| `Média` \| `Grande` |
| `uf` | STRING | SIM | — | Sigla do estado (2 letras, ex.: `SP`, `MG`) |
| `data_inicio_relacionamento` | DATE | SIM | — | Data de início do relacionamento comercial |

**Nota:** 1 duplicata de `empresa_id=8` foi detectada na camada RAW e removida no MERGE incremental da silver.

---

### dim_plano

> **Grão:** 1 linha por plano de saúde  
> **Fonte:** `silver_planos`  
> **Registros:** ~28  

| Coluna | Tipo | Nulável | PK/FK | Descrição |
|---|---|---|---|---|
| `plano_id` | INT | NÃO | PK | Chave primária do plano |
| `operadora_id` | INT | SIM | FK → `dim_operadora` | Operadora que comercializa o plano |
| `plano_nome` | STRING | SIM | — | Nome comercial do plano |
| `segmentacao` | STRING | SIM | — | Tipo de cobertura: `Ambulatorial` \| `Hospitalar` \| `Ambulatorial + Hospitalar` \| `Hospitalar + Obstetrícia` |
| `acomodacao` | STRING | SIM | — | `Enfermaria` \| `Apartamento` |
| `coparticipacao` | STRING | SIM | — | Possui coparticipação: `Sim` \| `Não` |
| `preco_vida_mes` | DECIMAL(10,2) | SIM | — | Preço por vida por mês em R$ |

---

### dim_corretor

> **Grão:** 1 linha por corretor de venda  
> **Fonte:** `silver_corretores`  
> **Registros:** ~25  

| Coluna | Tipo | Nulável | PK/FK | Descrição |
|---|---|---|---|---|
| `corretor_id` | INT | NÃO | PK | Chave primária do corretor |
| `corretor_nome` | STRING | SIM | — | Nome completo do corretor |
| `regiao` | STRING | SIM | — | Região de atuação: `Norte` \| `Nordeste` \| `Centro-Oeste` \| `Sudeste` \| `Sul` |
| `senioridade` | STRING | SIM | — | `Júnior` \| `Pleno` \| `Sênior` |
| `data_admissao` | DATE | SIM | — | Data de admissão do corretor |

---

### dim_data

> **Grão:** 1 linha por dia do calendário  
> **Fonte:** Gerada sinteticamente via `explode(sequence(...))`  
> **Range:** 2018-01-01 → 2030-12-31  
> **Registros:** 4.748  

`dim_data` é uma **dimensão conformada** — conecta-se a múltiplas colunas de data nas tabelas fato sem necessidade de colunas FK explícitas nas facts.

| Coluna | Tipo | Nulável | Descrição |
|---|---|---|---|
| `data_id` | INT | NÃO | **PK inteira** no formato YYYYMMDD (ex.: `20240315`). Útil para ferramentas de BI que preferem chaves inteiras |
| `data` | DATE | SIM | **PK natural** (DATE). Use esta coluna nos JOINs com as facts |
| `ano` | INT | SIM | Ano (ex.: 2024) |
| `trimestre` | INT | SIM | Trimestre: `1` a `4` |
| `semestre` | INT | SIM | `1` = Jan–Jun \| `2` = Jul–Dez |
| `mes` | INT | SIM | Mês: `1` a `12` |
| `dia` | INT | SIM | Dia do mês: `1` a `31` |
| `semana_ano` | INT | SIM | Semana ISO do ano: `1` a `53` |
| `dia_semana` | INT | SIM | `1`=Dom, `2`=Seg, `3`=Ter, `4`=Qua, `5`=Qui, `6`=Sex, `7`=Sáb |
| `nome_dia_semana` | STRING | SIM | Nome por extenso em PT-BR (ex.: `Segunda-feira`) |
| `nome_mes` | STRING | SIM | Nome por extenso em PT-BR (ex.: `Março`) |
| `ano_mes` | STRING | SIM | Período no formato `YYYY-MM` (ex.: `2024-03`). Útil para `GROUP BY` mensal |
| `ano_trimestre` | STRING | SIM | Período no formato `YYYY-QN` (ex.: `2024-Q2`). Útil para `GROUP BY` trimestral |
| `is_fim_semana` | BOOLEAN | SIM | `TRUE` se Sábado ou Domingo |
| `ultimo_dia_mes` | DATE | SIM | Último dia do mês. Útil para filtros de fechamento mensal |

**Conexões com as tabelas fato:**

| Fact | Coluna da fact | JOIN |
|---|---|---|
| `fat_contratos` | `data_venda` | `fat_contratos.data_venda = dim_data.data` |
| `fat_contratos` | `vigencia_inicio` | `fat_contratos.vigencia_inicio = dim_data.data` |
| `fat_contratos` | `vigencia_fim` | `fat_contratos.vigencia_fim = dim_data.data` |
| `fat_utilizacao` | `data_evento` | `fat_utilizacao.data_evento = dim_data.data` |
| `fat_sinistralidade` | `ano_mes` | `fat_sinistralidade.ano_mes = dim_data.ano_mes` |

---

### dim_beneficiario

> **Grão:** 1 linha por beneficiário  
> **Fonte:** `silver_beneficiarios`  
> **Registros:** ~1.558  

| Coluna | Tipo | Nulável | PK/FK | Descrição |
|---|---|---|---|---|
| `beneficiario_id` | INT | NÃO | PK | Chave primária do beneficiário |
| `contrato_id` | INT | SIM | FK → `fat_contratos` | Contrato ao qual o beneficiário pertence |
| `empresa_id` | INT | SIM | FK → `dim_empresa` | Empresa empregadora do beneficiário |
| `data_nascimento` | DATE | SIM | — | Data de nascimento |
| `sexo` | STRING | SIM | — | `Masculino` \| `Feminino` \| NULL (~19% nulos na fonte) |
| `tipo_beneficiario` | STRING | SIM | — | `Titular` \| `Dependente` |
| `data_adesao` | DATE | SIM | — | Data de adesão ao plano |
| `data_cancelamento` | DATE | SIM | — | Data de cancelamento; `NULL` = beneficiário ativo (~89% nulos) |
| `situacao` | STRING | NÃO | — | **(derivado)** `Ativo` (quando `data_cancelamento IS NULL`) \| `Cancelado` |
| `faixa_etaria` | STRING | SIM | — | **(derivado)** Faixa etária calculada em relação a `CURRENT_DATE()`: `00-17` \| `18-29` \| `30-44` \| `45-59` \| `60+` |

**Atenção:** `faixa_etaria` é calculada em tempo de execução do notebook. Reexecutar em datas diferentes pode alterar a classificação de beneficiários próximos das faixas limítrofes.

---

## Fatos

### fat_contratos

> **Grão:** 1 linha por contrato de plano de saúde  
> **Fonte:** `silver_contratos`  
> **Registros:** ~109  

| Coluna | Tipo | Nulável | PK/FK | Descrição |
|---|---|---|---|---|
| `contrato_id` | INT | NÃO | PK | Chave primária do contrato |
| `empresa_id` | INT | SIM | FK → `dim_empresa` | Empresa contratante |
| `plano_id` | INT | SIM | FK → `dim_plano` | Plano contratado |
| `corretor_id` | INT | SIM | FK → `dim_corretor` | Corretor responsável pela venda |
| `data_venda` | DATE | SIM | FK → `dim_data` | Data de fechamento da venda |
| `vigencia_inicio` | DATE | SIM | FK → `dim_data` | Início da vigência contratual |
| `vigencia_fim` | DATE | SIM | FK → `dim_data` | Fim da vigência contratual |
| `num_vidas` | INT | SIM | — | Quantidade de vidas cobertas |
| `valor_mensal` | DECIMAL(15,2) | SIM | — | Valor mensal total do contrato em R$ (1 contrato com NULL na fonte) |
| `status` | STRING | SIM | — | `Ativo` \| `Cancelado` \| `Renovado` |
| `receita_total_mensal` | DECIMAL | NÃO | — | **(derivado)** `COALESCE(valor_mensal * num_vidas, 0)` |
| `dias_vigencia` | INT | SIM | — | **(derivado)** `DATEDIFF(vigencia_fim, vigencia_inicio)` |

---

### fat_utilizacao

> **Grão:** 1 linha por evento de utilização (sinistro)  
> **Fonte:** `silver_utilizacao` + `silver_beneficiarios` + `silver_contratos`  
> **Registros:** ~6.480  

As colunas `contrato_id`, `empresa_id` e `plano_id` são enriquecidas por JOINs em tempo de build (não existem na fonte original `silver_utilizacao`) para eliminar JOINs em tempo de query.

| Coluna | Tipo | Nulável | PK/FK | Descrição |
|---|---|---|---|---|
| `evento_id` | INT | NÃO | PK | Chave primária do evento |
| `beneficiario_id` | INT | SIM | FK → `dim_beneficiario` | Beneficiário que gerou o evento |
| `contrato_id` | INT | SIM | FK → `fat_contratos` | **(derivado via beneficiario)** Contrato do beneficiário |
| `empresa_id` | INT | SIM | FK → `dim_empresa` | **(derivado via beneficiario)** Empresa do beneficiário |
| `plano_id` | INT | SIM | FK → `dim_plano` | **(derivado via contrato)** Plano do contrato |
| `data_evento` | DATE | SIM | FK → `dim_data` | Data de ocorrência do evento médico |
| `tipo_evento` | STRING | SIM | — | `Consulta` \| `Exame` \| `Cirurgia` \| `Pronto-socorro` \| `Internação` \| outros (lista aberta) |
| `especialidade` | STRING | SIM | — | Especialidade médica (ex.: Cardiologia, Oncologia, Dermatologia) |
| `valor_sinistro` | DECIMAL(12,2) | SIM | — | Custo do evento em R$ |

---

### fat_sinistralidade

> **Grão:** 1 linha por empresa × mês calendário  
> **Fonte:** `gold_fat_contratos` + `gold_fat_utilizacao` + `gold_dim_empresa`  
> **Registros:** ~985 (empresa × meses com receita ou sinistro)  
> **PK composta:** `(empresa_id, ano_mes)`  

Tabela pré-agregada e **pronta para consumo direto em BI** — atributos de `dim_empresa` são desnormalizados para eliminar JOINs nas ferramentas de visualização.

| Coluna | Tipo | Nulável | PK/FK | Descrição |
|---|---|---|---|---|
| `empresa_id` | INT | SIM | PK + FK → `dim_empresa` | Empresa (componente da PK) |
| `ano_mes` | STRING | SIM | PK + FK → `dim_data.ano_mes` | Período no formato `YYYY-MM` (componente da PK) |
| `empresa_nome` | STRING | SIM | — | **(desnorm.)** Nome da empresa |
| `setor` | STRING | SIM | — | **(desnorm.)** Setor econômico |
| `porte` | STRING | SIM | — | **(desnorm.)** Porte da empresa |
| `uf` | STRING | SIM | — | **(desnorm.)** Estado |
| `ano` | INT | SIM | — | **(derivado)** Ano extraído de `ano_mes` |
| `mes` | INT | SIM | — | **(derivado)** Mês extraído de `ano_mes` |
| `receita_premios` | DECIMAL | SIM | — | **[P1]** Soma de `valor_mensal` dos contratos com vigência ativa no mês |
| `custo_sinistros` | DECIMAL | SIM | — | **[P2]** Soma de `valor_sinistro` por `data_evento` no mês |
| `margem_tecnica` | DECIMAL | SIM | — | **(derivado)** `receita_premios − custo_sinistros`. Positivo = margem favorável |
| `sinistralidade` | DECIMAL | SIM | — | **(derivado)** `custo_sinistros / receita_premios` (ratio, 4 casas). `NULL` quando `receita_premios = 0` **[P4]** |
| `sinistralidade_pct` | DECIMAL | SIM | — | **(derivado)** `sinistralidade × 100` (percentual, 2 casas). `NULL` quando `receita_premios = 0` **[P4]** |
| `vidas_ativas` | BIGINT | SIM | — | Total de vidas nos contratos ativos no mês |
| `contratos_ativos` | BIGINT | SIM | — | Quantidade de contratos ativos no mês |
| `qtd_eventos` | BIGINT | SIM | — | Quantidade de eventos de utilização no mês |
| `ticket_medio_evento` | DECIMAL | SIM | — | **(derivado)** `custo_sinistros / qtd_eventos` |
| `custo_per_capita` | DECIMAL | SIM | — | **(derivado)** `custo_sinistros / vidas_ativas` |
| `frequencia_utilizacao` | DOUBLE | SIM | — | **(derivado)** `qtd_eventos / vidas_ativas` (eventos por vida/mês) |
| `sinistro_sem_receita` | BOOLEAN | SIM | — | **Flag [P4]** `TRUE` = sinistro registrado sem receita correspondente |

---

## Premissas de Cálculo — fat_sinistralidade

### [P1] Receita de Prêmios (denominador)

- **Fonte:** `gold_fat_contratos.valor_mensal`
- **Regra:** Um contrato contribui com o valor `valor_mensal` **integral** para cada mês calendário em que sua vigência (`vigencia_inicio` ↔ `vigencia_fim`) contém pelo menos 1 dia.
- **Contratos parciais:** início ou fim no meio do mês recebem receita integral — **sem pro-rata**.
  - Impacto: tende a **subestimar** a sinistralidade nos meses de início e encerramento do contrato.
- **Exclusão:** contratos com `valor_mensal IS NULL` são excluídos do denominador (1 contrato identificado na fonte).
- **Implementação:** `LATERAL VIEW explode(sequence(date_trunc('MONTH', vigencia_inicio), date_trunc('MONTH', vigencia_fim), INTERVAL 1 MONTH))`

### [P2] Custo de Sinistros (numerador)

- **Fonte:** `gold_fat_utilizacao.valor_sinistro`
- **Critério de competência:** `data_evento` — data em que o evento médico ocorreu.
- **Inclui:** todos os tipos de evento registrados (Consulta, Exame, Cirurgia, Pronto-socorro, Internação e demais).
- **Não inclui:** reserva IBNR (*Incurred But Not Reported* — sinistros ocorridos e ainda não reportados).

### [P3] Meses com Receita e sem Sinistro

- `custo_sinistros = 0` → `sinistralidade = 0.0000` (0,00%)
- Representa meses em que nenhum evento foi gerado por beneficiários da empresa.

### [P4] Meses com Sinistro e sem Receita

- `receita_premios = 0` → `sinistralidade = NULL`
- `sinistro_sem_receita = TRUE` — flag para monitoramento
- **Causa provável:** evento de beneficiário cujo contrato foi cancelado retroativamente, ou data de vigência inconsistente na fonte.
- **Ação recomendada:** investigar registros com este flag ativo.

### [P5] Janela Temporal

- Gerada **dinamicamente** pela união dos meses de vigência dos contratos e dos meses com eventos de sinistro.
- Não há janela fixa de data — o range acompanha os dados disponíveis.

---

## Linhagem de Dados

```
RAW                     SILVER                      GOLD
──────────────────────────────────────────────────────────────────────────────
raw_operadoras      ──► silver_operadoras       ──► gold_dim_operadora
raw_empresas        ──► silver_empresas         ──► gold_dim_empresa
raw_planos          ──► silver_planos           ──► gold_dim_plano
raw_corretores      ──► silver_corretores       ──► gold_dim_corretor
[sintético]                                    ──► gold_dim_data
raw_beneficiarios   ──► silver_beneficiarios   ──► gold_dim_beneficiario
raw_contratos       ──► silver_contratos       ──► gold_fat_contratos
raw_utilizacao      ──► silver_utilizacao  ┐
raw_beneficiarios   ──► silver_beneficiarios├──► gold_fat_utilizacao
raw_contratos       ──► silver_contratos   ┘
gold_fat_contratos  ┐
gold_fat_utilizacao ├──────────────────────────► gold_fat_sinistralidade
gold_dim_empresa    ┘
```

**Padrão de atualização das silver:** MERGE incremental via `DeltaTable.merge()` com `dropDuplicates([pk])` antes do merge.

**Padrão de atualização das gold:** `CREATE OR REPLACE TABLE ... AS SELECT ...` (full refresh a cada execução do notebook).

---

## Guia de JOINs

### Consultar sinistralidade com atributos de empresa e plano

```sql
SELECT
  s.empresa_nome,
  s.setor,
  s.porte,
  d.nome_mes,
  d.ano,
  s.sinistralidade_pct,
  s.receita_premios,
  s.custo_sinistros
FROM homologacao.salutar_saude.gold_fat_sinistralidade s
LEFT JOIN homologacao.salutar_saude.gold_dim_data d
  ON s.ano_mes = d.ano_mes
WHERE d.ano = 2025
ORDER BY s.sinistralidade_pct DESC NULLS LAST;
```

### Eventos de utilização com todos os atributos dimensionais

```sql
SELECT
  u.evento_id,
  b.tipo_beneficiario,
  b.faixa_etaria,
  b.sexo,
  e.empresa_nome,
  e.porte,
  p.plano_nome,
  p.segmentacao,
  op.operadora_nome,
  d.nome_mes,
  d.trimestre,
  u.tipo_evento,
  u.especialidade,
  u.valor_sinistro
FROM homologacao.salutar_saude.gold_fat_utilizacao    u
JOIN homologacao.salutar_saude.gold_dim_beneficiario  b  ON u.beneficiario_id = b.beneficiario_id
JOIN homologacao.salutar_saude.gold_dim_empresa        e  ON u.empresa_id      = e.empresa_id
JOIN homologacao.salutar_saude.gold_fat_contratos      c  ON u.contrato_id     = c.contrato_id
JOIN homologacao.salutar_saude.gold_dim_plano          p  ON u.plano_id        = p.plano_id
JOIN homologacao.salutar_saude.gold_dim_operadora      op ON p.operadora_id    = op.operadora_id
JOIN homologacao.salutar_saude.gold_dim_data           d  ON u.data_evento     = d.data;
```

### Receita mensal por operadora (via plano → operadora)

```sql
SELECT
  op.operadora_nome,
  d.ano_mes,
  SUM(c.receita_total_mensal)  AS receita_total,
  SUM(c.num_vidas)             AS total_vidas
FROM homologacao.salutar_saude.gold_fat_contratos c
JOIN homologacao.salutar_saude.gold_dim_plano      p  ON c.plano_id     = p.plano_id
JOIN homologacao.salutar_saude.gold_dim_operadora  op ON p.operadora_id = op.operadora_id
JOIN homologacao.salutar_saude.gold_dim_data        d  ON c.data_venda   = d.data
GROUP BY op.operadora_nome, d.ano_mes
ORDER BY op.operadora_nome, d.ano_mes;
```

### Sinistralidade acumulada YTD por empresa

```sql
SELECT
  empresa_nome,
  ano,
  ROUND(
    SUM(custo_sinistros) / NULLIF(SUM(receita_premios), 0) * 100, 2
  ) AS sinistralidade_ytd_pct,
  SUM(receita_premios)  AS receita_ytd,
  SUM(custo_sinistros)  AS custo_ytd,
  SUM(margem_tecnica)   AS margem_ytd
FROM homologacao.salutar_saude.gold_fat_sinistralidade
GROUP BY empresa_nome, ano
ORDER BY ano DESC, sinistralidade_ytd_pct DESC NULLS LAST;
```

---

## Exemplos de Queries

### Top 10 especialidades por custo total

```sql
SELECT
  especialidade,
  tipo_evento,
  COUNT(*)               AS qtd_eventos,
  ROUND(SUM(valor_sinistro), 2)  AS custo_total,
  ROUND(AVG(valor_sinistro), 2)  AS ticket_medio
FROM homologacao.salutar_saude.gold_fat_utilizacao
GROUP BY especialidade, tipo_evento
ORDER BY custo_total DESC
LIMIT 10;
```

### Empresas com sinistralidade > 100% no último mês disponível

```sql
WITH ultimo_mes AS (
  SELECT MAX(ano_mes) AS ano_mes
  FROM homologacao.salutar_saude.gold_fat_sinistralidade
)
SELECT
  s.empresa_nome,
  s.setor,
  s.porte,
  s.uf,
  s.receita_premios,
  s.custo_sinistros,
  s.sinistralidade_pct,
  s.vidas_ativas
FROM homologacao.salutar_saude.gold_fat_sinistralidade s
JOIN ultimo_mes m ON s.ano_mes = m.ano_mes
WHERE s.sinistralidade_pct > 100
ORDER BY s.sinistralidade_pct DESC;
```

### Evolução mensal da sinistralidade média da carteira

```sql
SELECT
  ano_mes,
  ano,
  mes,
  ROUND(SUM(custo_sinistros) / NULLIF(SUM(receita_premios), 0) * 100, 2) AS sinistralidade_carteira_pct,
  SUM(receita_premios)   AS receita_total,
  SUM(custo_sinistros)   AS custo_total,
  SUM(vidas_ativas)      AS total_vidas,
  SUM(qtd_eventos)       AS total_eventos
FROM homologacao.salutar_saude.gold_fat_sinistralidade
WHERE sinistro_sem_receita = FALSE  -- exclui registros com problema de dados
GROUP BY ano_mes, ano, mes
ORDER BY ano_mes;
```

### Comparativo de faixa etária × tipo de evento

```sql
SELECT
  b.faixa_etaria,
  u.tipo_evento,
  COUNT(*)                          AS qtd,
  ROUND(SUM(u.valor_sinistro), 2)   AS custo_total,
  ROUND(AVG(u.valor_sinistro), 2)   AS ticket_medio
FROM homologacao.salutar_saude.gold_fat_utilizacao   u
JOIN homologacao.salutar_saude.gold_dim_beneficiario b
  ON u.beneficiario_id = b.beneficiario_id
GROUP BY b.faixa_etaria, u.tipo_evento
ORDER BY b.faixa_etaria, custo_total DESC;
```

---

*Documentação gerada automaticamente a partir dos notebooks da camada Gold do projeto Salutar Saúde.*
