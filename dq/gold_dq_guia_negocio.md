# Guia de Qualidade de Dados — Gold Layer Salutar Saúde

> **Para quem é este documento?**
> Gestores, analistas de negócio e times de produto que consomem os dados da camada Gold. O objetivo é explicar, em linguagem de negócio, **o que é verificado, por que cada verificação importa e o que significa quando algo falha**.

---

## O que é a Camada Gold?

A camada Gold é o conjunto de tabelas prontas para análise e geração de relatórios. É a "versão oficial" dos dados da Salutar Saúde — os números que aparecem em dashboards, relatórios financeiros e indicadores de performance.

Antes de qualquer relatório ser gerado, um conjunto de **74 verificações automáticas** é executado para garantir que os dados são confiáveis. Esse processo é chamado de **Suíte de Qualidade de Dados (DQ Suite)**.

---

## O que Acontece se Algo Falhar?

Cada verificação pode ter três resultados:

| Resultado | Significado |
|---|---|
| ✅ **PASS** | Tudo certo. O dado está íntegro e confiável. |
| ⚠️ **WARN** | Atenção. Algo fora do padrão foi detectado, mas pode ser esperado. Investigue. |
| ❌ **FAIL** | Problema crítico. O dado **não deve ser utilizado** até que seja corrigido. |

Um FAIL bloqueia automaticamente o pipeline e impede que dados incorretos cheguem aos relatórios.

---

## As 9 Tabelas Verificadas

A suíte cobre todas as tabelas da Gold, divididas em **dimensões** (tabelas de referência) e **fatos** (tabelas de eventos e métricas).

### Tabelas de Referência (Dimensões)

| Tabela | O que representa |
|---|---|
| **Operadoras** | As operadoras de saúde com as quais a Salutar trabalha (ex.: Unimed, Bradesco Saúde) |
| **Empresas** | As empresas clientes que contrataram planos coletivos |
| **Planos** | O catálogo de planos disponíveis por operadora, com preço e tipo de cobertura |
| **Corretores** | Os corretores de venda responsáveis pelos contratos |
| **Beneficiários** | Cada pessoa (titular ou dependente) coberta por um plano |
| **Calendário** | Tabela de datas de 2018 a 2030, usada para análises por período |

### Tabelas de Fatos (Eventos e Métricas)

| Tabela | O que representa |
|---|---|
| **Contratos** | Cada contrato de plano firmado entre uma empresa e a Salutar |
| **Utilização** | Cada evento de saúde gerado pelos beneficiários (consultas, exames, cirurgias etc.) |
| **Sinistralidade** | Resumo financeiro mensal por empresa: receita de prêmios, custo de sinistros e margem técnica |

---

## O que é Verificado em Cada Tabela

### 1. Operadoras

**Por que importa:** Se uma operadora estiver duplicada, relatórios de performance por operadora vão inflar os números, dando a impressão de mais negócios do que o real.

| Verificação | O que garante |
|---|---|
| Nenhuma operadora duplicada | Cada operadora aparece exatamente uma vez |
| Nenhuma operadora sem nome | Todos os registros têm nome preenchido |
| Pelo menos 1 operadora cadastrada | A tabela não está vazia |

---

### 2. Empresas Clientes

**Por que importa:** Empresas duplicadas distorcem métricas de carteira, como número de clientes ativos, receita por empresa e análise de churn.

| Verificação | O que garante |
|---|---|
| Nenhuma empresa duplicada | Cada CNPJ/empresa aparece exatamente uma vez |
| Nome da empresa sempre preenchido | Não há registros anônimos |
| Porte dentro do padrão | Classificação correta: Pequena, Média ou Grande |
| UF com 2 caracteres | Sigla de estado válida (ex.: SP, RJ, MG) |
| Data de início do relacionamento não está no futuro | Datas coerentes com a realidade |

---

### 3. Planos

**Por que importa:** Um plano sem operadora válida é um dado órfão — ele não pode ser rastreado até a operadora responsável, o que quebra relatórios de rentabilidade por operadora.

| Verificação | O que garante |
|---|---|
| Nenhum plano duplicado | Cada plano aparece exatamente uma vez |
| Toda operadora referenciada existe | Nenhum plano aponta para operadora inexistente |
| Tipo de acomodação válido | Somente: Enfermaria ou Apartamento |
| Coparticipação válida | Somente: Sim ou Não |
| Preço de vida por mês positivo | Nenhum plano com preço zero ou negativo |

---

### 4. Corretores

**Por que importa:** Comissões e metas de corretores dependem de dados corretos. Um corretor com nível de senioridade inválido pode gerar erros no cálculo de comissão.

| Verificação | O que garante |
|---|---|
| Nenhum corretor duplicado | Cada corretor aparece exatamente uma vez |
| Nível de senioridade válido | Somente: Júnior, Pleno ou Sênior |
| Região preenchida | Nenhum corretor sem região de atuação |
| Data de admissão não está no futuro | Datas coerentes com a realidade |

---

### 5. Beneficiários

**Por que importa:** Beneficiários são a base do cálculo de vidas ativas — o principal indicador de tamanho de carteira. Qualquer erro aqui afeta diretamente as métricas de saúde coletiva.

| Verificação | O que garante |
|---|---|
| Nenhum beneficiário duplicado | Cada beneficiário aparece exatamente uma vez |
| Todo beneficiário vinculado a um contrato existente | Nenhum beneficiário "solto", sem contrato |
| Todo beneficiário vinculado a uma empresa existente | Integridade da cadeia empresa → contrato → beneficiário |
| Tipo de beneficiário válido | Somente: Titular ou Dependente |
| Status de situação válido | Somente: Ativo ou Cancelado |
| Faixa etária válida | Dentro das faixas: 00-17, 18-29, 30-44, 45-59, 60+ |
| Situação coerente com data de cancelamento | Se tem data de cancelamento → Cancelado. Se não tem → Ativo |
| Data de nascimento não está no futuro | Datas coerentes com a realidade |

---

### 6. Contratos

**Por que importa:** O contrato é o elo central do modelo. Um contrato duplicado dobra o valor de receita calculado. Um contrato com datas invertidas pode gerar vigência negativa, corrompendo qualquer análise temporal.

| Verificação | O que garante |
|---|---|
| Nenhum contrato duplicado | Cada contrato aparece exatamente uma vez |
| Empresa, plano e corretor do contrato existem | Integridade referencial completa |
| Status do contrato válido | Somente: Ativo, Cancelado ou Renovado |
| Número de vidas positivo | Nenhum contrato com zero vidas |
| Data de fim de vigência após data de início | Nenhum contrato com vigência invertida |
| Receita mensal = valor por vida × número de vidas | Cálculo financeiro validado |
| Dias de vigência calculados corretamente | Consistência matemática da duração do contrato |

---

### 7. Utilização (Eventos de Saúde)

**Por que importa:** Cada linha representa um sinistro real — uma consulta, exame ou cirurgia realizada por um beneficiário. Duplicatas inflam o custo de sinistros e distorcem o índice de sinistralidade. Eventos sem vínculo com beneficiário ou contrato são "gastos fantasma".

| Verificação | O que garante |
|---|---|
| Nenhum evento duplicado | Cada sinistro contabilizado apenas uma vez |
| Beneficiário, contrato, empresa e plano existem | Nenhum sinistro "órfão" |
| Data do evento sempre preenchida | Nenhum sinistro sem data |
| Valor do sinistro sempre preenchido e positivo | Nenhum sinistro sem custo ou com custo negativo |
| Data do evento dentro do calendário disponível | Evento dentro do range 2018–2030 |
| ⚠️ Tipo de evento pode incluir categorias novas | A lista de tipos é aberta — novos tipos são esperados e aceitos |

---

### 8. Sinistralidade Mensal

**Por que importa:** Esta é a tabela de **indicadores financeiros** — o coração dos relatórios de saúde coletiva. O índice de sinistralidade (custo de sinistros ÷ receita de prêmios) determina se a carteira está saudável ou em prejuízo. Qualquer erro aqui compromete diretamente as decisões de precificação, renovação e risco.

#### Regras de Negócio Aplicadas

A sinistralidade é calculada com base em 4 premissas de negócio documentadas:

| Premissa | Descrição |
|---|---|
| **[P1] Receita de prêmios** | Soma do valor mensal de todos os contratos ativos no mês. Contratos com valor nulo são excluídos. |
| **[P2] Custo de sinistros** | Soma dos valores dos eventos de saúde com data no mês. Sem projeção de IBNR (sinistros ainda não avisados). |
| **[P3] Margem técnica** | Receita menos custo. Positivo indica margem favorável; negativo indica prejuízo técnico. |
| **[P4] Sinistralidade indefinida** | Quando não há receita no mês (receita = R$ 0), o índice é indefinido (NULL). Um flag `sinistro_sem_receita` identifica empresas com custo mas sem receita — situação que requer investigação imediata. |

#### Verificações Aplicadas

| Verificação | O que garante |
|---|---|
| Nenhuma combinação empresa + mês duplicada | Cada empresa tem exatamente um registro por mês |
| Toda empresa referenciada existe | Nenhum registro de sinistralidade para empresa inexistente |
| Todo mês referenciado existe no calendário | Consistência temporal |
| Receita e custo nunca negativos | Valores financeiros coerentes |
| Sinistralidade calculada quando há receita [P1] | Nenhum índice em branco quando deveria estar preenchido |
| Sinistralidade indefinida quando não há receita [P4] | Respeito à premissa de cálculo |
| Margem técnica = receita − custo [P3] | Consistência matemática |
| Sinistralidade (%) = índice × 100 | Consistência entre as duas formas de exibição |
| Flag `sinistro_sem_receita` coerente com os valores [P4] | O indicador de alerta está correto |

---

## Resumo dos 74 Testes por Categoria

| Categoria | Qtd | O que representa |
|---|---:|---|
| Domínio | 20 | Valores dentro das listas e regras aceitas |
| Regras de Negócio | 16 | Cálculos e lógicas financeiras corretas |
| Integridade | 14 | Vínculos entre tabelas sem registros órfãos |
| Unicidade | 10 | Chaves primárias sem duplicatas |
| Volume | 9 | Tabelas com quantidade esperada de registros |
| Completude | 5 | Campos obrigatórios sem valores em branco |

---

## Histórico de Execuções

| Data | Total | PASS | WARN | FAIL | Status |
|---|---|---|---|---|---|
| 04/07/2026 (execução 1) | 74 | 69 | 1 | 4 | ❌ Com falhas |
| 04/07/2026 (execução 2) | 74 | 73 | 1 | 0 | ✅ Aprovado |

**Causa das falhas na execução 1:** Duplicatas originadas na camada Silver (empresa_id e contrato_id duplicados) propagaram-se até a Gold, gerando contagem incorreta de eventos e de sinistralidade mensal. Corrigido com deduplicação defensiva aplicada na reconstrução das tabelas Gold.

---

## O que Fazer em Caso de Falha

1. **Leia o detalhe da falha** — a mensagem indica exatamente qual tabela, qual regra e quantos registros estão com problema.
2. **Não use os dados para tomada de decisão** enquanto houver FAILs ativos.
3. **Acione o time de Engenharia de Dados** com o relatório da suíte — o arquivo `gold_dq_tests` no workspace da Salutar contém o detalhamento completo.
4. Após a correção, **aguarde a re-execução** da suíte para confirmar que todos os testes voltaram a PASS.

---

## Glossário

| Termo | Definição |
|---|---|
| **Sinistralidade** | Proporção do custo de saúde em relação à receita de prêmios. Acima de 100% significa que a operação está no prejuízo técnico. |
| **Prêmio** | Valor pago mensalmente pela empresa para manter os beneficiários cobertos pelo plano. |
| **Sinistro** | Qualquer evento de saúde gerado por um beneficiário (consulta, exame, internação etc.) que gera custo para a operadora. |
| **Vigência** | Período de validade de um contrato, definido por data de início e data de fim. |
| **Titular / Dependente** | Titular é o funcionário contratante; dependentes são seus familiares cobertos pelo mesmo plano. |
| **IBNR** | *Incurred But Not Reported* — sinistros já ocorridos mas ainda não registrados. A sinistralidade calculada aqui **não inclui** projeção de IBNR (premissa P2). |
| **Chave primária** | Identificador único de cada registro em uma tabela. A duplicação de chaves é sempre um erro crítico. |
| **Integridade referencial** | Garantia de que todo vínculo entre tabelas aponta para um registro que realmente existe. |

---

*Documento gerado automaticamente a partir da suíte `gold_dq_tests` — Salutar Saúde Data Lake, `homologacao.salutar_saude`.*
