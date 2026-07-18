# Definição do Modelo — Segmentação de Consumidores (Squad 3, Lojas Físicas)

> Documento de definição técnica para a etapa de IA/ML do projeto de estágio. Cobre os modelos escolhidos, o motivo de cada um e as tabelas de origem necessárias.

---

## 1. Problema de negócio

**Tema:** Segmentação de Consumidores
**Problema:** Identificar os tipos de compradores e seus padrões de compra.

O problema tem duas partes distintas, e cada uma exige um tipo de modelo diferente:

1. **"Tipos de compradores"** → agrupar clientes por perfil de comportamento → **clustering**.
2. **"Padrões de compra"** → identificar relações entre produtos comprados juntos → **regras de associação**.

Por isso a solução combina duas trilhas de modelagem em paralelo, e não um único algoritmo.

**Restrição de escopo definida:** não usar RFV (Recência/Frequência/Valor), não usar Box-Cox, não usar K-Means. Os modelos abaixo foram escolhidos para atender a essa restrição.

---

## 1.1 Achado de negócio — recorrência de cliente identificado

Ao executar a Feature Engineering sobre a base real (Silver `physical_vendas_caixa` +
`physical_itens_venda_caixa`, todo o período disponível), o resultado revelou um fato de
negócio que muda a estratégia de segmentação:

| Métrica | Valor |
|---|---|
| Total de clientes com `cpf_cliente` identificado | 111.826 |
| Clientes com **apenas 1 transação** no período (`qtd_transacoes = 1`) | ~107.120 (**≈ 96%**) |
| Clientes **recorrentes** (`qtd_transacoes > 1`) | ~4.706 (**≈ 4%**) |

**Implicação:** para 96% da base, features de comportamento temporal (`dias_entre_compras_media`,
`desvio_padrao_ticket`) ficam nulas ou degeneradas (`qtd_lojas_distintas = 1`,
`pct_forma_pagamento_dominante = 100%` sempre) — não discriminam nada entre clientes. Rodar
um único modelo de clustering com essas features misturado ao restante tende a produzir um
cluster gigante e pouco informativo de "compradores únicos", diluindo o valor da segmentação.

**Não é uma falha de tratamento de dado** — é a taxa real de identificação de cliente em
loja física, coerente com o que a regra de negócio de `physical_vendas_caixa` já sinalizava
(% de vendas com CPF identificado é tratado como métrica de engajamento do programa de
fidelidade, não como 100% esperado). Pode ser, inclusive, um insight acionável para o
negócio: hoje só ~4% da base permite qualquer análise de recorrência, o que sugere espaço
para uma campanha de incentivo à identificação de CPF no caixa.

**Decisão de modelagem — segmentação em duas camadas:**

1. **Camada A — Perfil (base completa, 111.826 clientes).** Segmenta por características que
   fazem sentido mesmo para compra única: ticket médio, categoria de produto dominante,
   diversidade de categoria, forma de pagamento, loja. Usa Clustering Hierárquico ou GMM
   (seção 2.1).
2. **Camada B — Comportamento temporal (apenas os ~4.706 clientes recorrentes).** Segmenta
   adicionalmente por frequência, regularidade de compra e evolução do ticket ao longo do
   tempo — dimensão só calculável para quem tem mais de uma compra.

Cada cliente recorrente recebe os dois segmentos (perfil + comportamento); cliente de compra
única recebe apenas o segmento de perfil. A coluna `flag_cliente_recorrente` (calculada em
`01_feature_engineering_clientes`) formaliza esse corte no próprio dataset de features, em vez
de deixar a decisão implícita num filtro solto no notebook de modelagem.

---

## 2. Modelos escolhidos

### 2.1 Clustering Hierárquico Aglomerativo — segmentação de clientes

Aplicado duas vezes, uma vez por camada (ver seção 1.1): Camada A com toda a base (features
de perfil) e Camada B só com clientes recorrentes (features de perfil + comportamento
temporal). São dois modelos independentes, não um modelo único com filtro.

**O que faz:** agrupa clientes em uma árvore de agrupamentos (dendrograma), unindo progressivamente os pares mais similares até formar um único grupo. Os segmentos finais são obtidos "cortando" a árvore na altura desejada.

**Motivo de uso:**
- Não exige definir o número de segmentos (K) antes de rodar o modelo — o dendrograma é gerado primeiro, e o corte é decidido depois, olhando a estrutura real dos dados e o que faz sentido para o negócio.
- Mais fácil de explicar visualmente para stakeholders não-técnicos do que centróides de K-Means.
- Evita a suposição de clusters esféricos de tamanho parecido, que é uma limitação conhecida do K-Means.
- Volume de dados do projeto (clientes com CPF identificado em lojas físicas) é compatível com o custo computacional do método.

**Alternativa/complemento — GMM (Gaussian Mixture Model):**
- Considerar se for útil mostrar que um cliente pode pertencer parcialmente a mais de um perfil (soft clustering, com probabilidade de pertencimento), em vez de um rótulo rígido de grupo.
- Validado via BIC/AIC, sem depender de Box-Cox no pré-processamento.

**Validação:** Silhouette Score (aplicável tanto a Hierárquico quanto a GMM, não é exclusivo de K-Means) + checagem de sentido de negócio (os grupos resultantes correspondem a perfis reais e acionáveis de cliente de loja física?).

---

### 2.2 Regras de Associação (Apriori / FP-Growth) — padrões de compra

**O que faz:** identifica combinações de produtos frequentemente comprados juntos na mesma transação, com métricas de suporte, confiança e lift (ex.: "quem compra X também compra Y").

**Motivo de uso:**
- Responde diretamente à segunda parte do problema de negócio ("padrões de compra"), que um modelo de clustering de clientes não cobre — clustering agrupa **pessoas**, regras de associação agrupam **produtos**.
- Não depende do resultado do clustering para rodar; pode ser executado em paralelo, direto sobre a base de itens vendidos.
- Resultado é diretamente acionável para o negócio (cross-sell, disposição de produtos na loja, promoções combinadas).

**Validação:** métricas de suporte mínimo e lift > 1 para reter apenas regras relevantes (evitar ruído de combinações aleatórias).

---

## 3. Tabelas de origem necessárias

| Tabela | Camada | Papel na segmentação | Observação |
|---|---|---|---|
| `physical_vendas_caixa` | **Silver** (ADLS, `abfss://squad3@internshipdatalake.dfs.core.windows.net/silver/physical_vendas_caixa`) — **375.000 linhas confirmadas** | Fonte do identificador do cliente (`cpf_cliente`), valor da venda, data, forma de pagamento, loja. Grão: 1 linha por transação. | Confirmada via `printSchema` + `.count()` reais. Mantida por outro colega da squad (consumida somente por leitura, nenhuma escrita). Já vem com tipos tratados: `id_loja`/`id_caixa`/`id_operador` (`integer`), `dt_venda` (`timestamp`), `valor_total_venda` (`decimal(10,2)`). Contém `cpf_cliente` (string, nullable) — nulo é esperado (cliente não se identifica em toda venda), não é erro de qualidade. **Requer agregação por `cpf_cliente` antes do clustering**, já que o grão é de transação, não de cliente. |
| `physical_itens_venda_caixa` | **Silver** (ADLS, `abfss://squad3@internshipdatalake.dfs.core.windows.net/silver/physical_itens_venda_caixa`) — **3.550.893 linhas confirmadas** | Detalhe de produto por transação — necessário para diversidade de categoria (feature de clustering) e para as regras de associação. Grão: 1 linha por item de venda. | Existência, volume e schema confirmados via `printSchema()` real. Colunas relevantes: `codigo_barras_produto`, `codigo_produto_normalizado`, `categoria_produto` (já vem pronta, sem precisar mapeamento manual), `quantidade`, `preco_unitario_registro`, `valor_item_analitico` (valor já validado, usar este — não `valor_total_item_original`, que é o valor bruto pré-validação). Join com `physical_vendas_caixa` via `id_transacao`. |
| `physical_lojas` | **Silver** (ADLS, `abfss://squad3@internshipdatalake.dfs.core.windows.net/silver/physical_lojas`) — **29 linhas confirmadas** | Contexto opcional de loja (cidade, estado, CNPJ), no grão de 1 linha por loja — mais granular que a Gold se for necessário cruzar direto com a transação. | Alternativa à `gold_physical_lojas` quando o contexto precisar estar no grão de loja (não de loja/mês). |
| `gold_physical_lojas` | Gold (SQL Server, schema `squad3`) — **870 linhas confirmadas** | Contexto opcional de loja/região (enriquecimento IBGE: população, renda per capita) para cruzar com o perfil do cliente. Grão: 1 linha por loja por mês (870 linhas / 29 lojas ≈ 30 meses de histórico). | Uso opcional, não obrigatório para a segmentação em si. Atenção ao grão (loja/mês) antes de fazer join direto com dado de cliente. |
| `gold_physical_feriado_dia` | Gold (SQL Server, schema `squad3`) — **26.433 linhas confirmadas** | Contexto opcional de feriado, para enriquecer o padrão de compra (ex.: cliente que compra mais em feriado). Grão: 1 linha por loja por dia. | Uso opcional. Existe **apenas em Gold** — não tem Bronze/Silver própria, é calculada a partir da Silver de itens dentro do próprio notebook Gold. |
| `gold_physical_clientes_segmentados` *(tabela nova, a criar)* | Gold (SQL Server, schema `squad3`) | Resultado final: 1 linha por `cpf_cliente`, com o segmento atribuído pelo clustering. | Grão e nome de tabela definidos para seguir o padrão das demais tabelas Gold do projeto. |

### Ponto de atenção sobre os dados de origem

- `cpf_cliente` é **nulo em parte das vendas**, mesmo já na Silver (cliente não é obrigado a se identificar no caixa — não é uma pendência de tratamento, é um comportamento real do negócio). É necessário medir a proporção real de nulos antes de definir a cobertura da segmentação — a análise final deve deixar explícito que os resultados cobrem apenas as vendas com cliente identificado.
- `physical_vendas_caixa` (Silver) é mantida por outro colega da squad — o notebook de segmentação deve **apenas ler** essa tabela (`spark.read`), nunca escrever nela, respeitando as regras de convivência do projeto.
- **Não existe regra técnica de validação de formato/duplicidade do `cpf_cliente`** na planilha de regras do projeto (só existe a regra de negócio sobre % de CPF identificado). Isso é um risco para a segmentação: se o CPF não estiver padronizado (ex.: `"123.456.789-00"` vs `"12345678900"`), o mesmo cliente pode ser contado como dois clientes diferentes, distorcendo os segmentos. Recomenda-se aplicar uma normalização (remover pontuação, validar 11 dígitos) no próprio notebook de preparação, antes de agregar por cliente.
- **Mismatch de granularidade confirmado:** `physical_vendas_caixa` e `physical_itens_venda_caixa` estão no grão de transação/item; o modelo de clustering exige grão de cliente. A agregação (`groupBy cpf_cliente`) é uma etapa obrigatória da preparação, não um detalhe implícito.

---

## 4. Schema confirmado por tabela

### `physical_vendas_caixa` (Silver)
```
id_transacao: string | id_loja: integer | id_caixa: integer | id_operador: integer
dt_venda: timestamp | valor_total_venda: decimal(10,2) | cpf_cliente: string (nullable)
tipo_pagamento: string | bronze_ingested_at: timestamp | bronze_source_file: string
silver_processed_at: timestamp | ano: integer | mes: integer
```

### `physical_itens_venda_caixa` (Silver)
```
id_item_venda: long | id_transacao: string | codigo_barras_produto: string
quantidade: double | preco_unitario_registro: decimal(10,2)
valor_total_item_original: decimal(10,2)  ← valor bruto, pré-validação
valor_item_analitico: double              ← valor validado, USAR ESTE nas features
codigo_produto_normalizado: string
categoria_produto: string  ← só 3 valores: perecivel / seco / NAO_CLASSIFICADO (não usar para diversidade)
flag_fk_invalido / flag_codigo_barras_ausente / flag_quantidade_invalido /
flag_preco_invalido / flag_valor_inconsistente: boolean  ← usar para filtrar antes de agregar
diferenca_valor: double | valor_calculado: double
bronze_source_file, bronze_ingested_at, silver_processed_at, dq_rules_version
```

### `ia/features_clientes` (Delta, dataset gerado — output da etapa de Feature Engineering)
```
cpf_cliente: string
ticket_medio, desvio_padrao_ticket, valor_total_gasto: double
qtd_transacoes: long | dias_entre_compras_media: double (nulo se qtd_transacoes = 1)
dias_desde_ultima_compra: integer
qtd_categorias_distintas: long  ← extraída de codigo_barras_produto, não de categoria_produto
qtd_itens_por_transacao_media: double | pct_itens_pereciveis: double
categoria_dominante: integer (label encoding) | categoria_dominante_nome: string
pct_compras_feriado, pct_forma_pagamento_dominante: double
qtd_lojas_distintas: long
flag_cliente_recorrente: boolean  ← qtd_transacoes > 1, formaliza o corte de Camada A/B
features_processed_at: timestamp | data_referencia_calculo: timestamp
```
**Confirmado por execução real:** 111.826 linhas (1 por cliente), gravado em
`abfss://squad3@internshipdatalake.dfs.core.windows.net/ia/features_clientes`.

### `physical_lojas` (Silver)
```
id_loja: long | nome_loja: string | cnpj: string | cnpj_tratado: string
cidade_loja: string | estado_loja: string | estado_tratado: string
peso_vendas: double
flag_cnpj_invalido / flag_cnpj_duplicado / flag_uf_invalido: boolean
bronze_source_file, bronze_ingested_at, silver_processed_at, dq_rules_version
```

### `gold_physical_feriado_dia` (SQL Server, schema `squad3`)
```
id_loja: bigint | ano: int | mes: int | trimestre: int | dt_venda_date: date
venda_em_feriado: bit | nome_feriado: nvarchar
receita_dia: float | qtd_transacoes_dia: bigint
```

### `gold_physical_lojas` (SQL Server, schema `squad3`)
```
id_loja: bigint | nome_loja: nvarchar | cidade_loja: nvarchar
estado_tratado: nvarchar | cnpj_tratado: nvarchar
populacao_cidade: bigint | renda_media_per_capita: float
ano: int | mes: int | semestre: int | ano_completo: bit
receita_mes: float | receita_total: float | receita_ano_anterior: float
crescimento_yoy_pct: float | crescimento_mom_pct: float
crescimento_mesmo_mes_ano_anterior: float | ticket_medio: float
qtd_transacoes: bigint
flag_cnpj_invalido / flag_uf_invalido / flag_sem_match_ibge: bit
```

**Observação sobre filtros de qualidade:** `physical_itens_venda_caixa` já traz flags de inconsistência prontas (`flag_fk_invalido`, `flag_quantidade_invalido`, `flag_preco_invalido`, `flag_valor_inconsistente`). O notebook de segmentação deve filtrar por essas flags (excluir onde `= true`) antes de agregar por cliente, para não herdar itens problemáticos já identificados pela Silver.

---

## 5. Resumo da abordagem

| Etapa | Modelo/Técnica | Substitui |
|---|---|---|
| Preparação | Feature engineering comportamental (ticket médio, dias entre compras, diversidade de categoria, % compra em feriado, forma de pagamento dominante) | RFV |
| Modelagem — clientes | Clustering Hierárquico Aglomerativo (ou GMM) | K-Means |
| Modelagem — produtos | Regras de Associação (Apriori / FP-Growth) | — (trilha nova, não substitui nada do RFV/K-Means) |
| Avaliação | Silhouette Score + validação de sentido de negócio | Elbow Method |
| Pré-processamento | Padronização direta das features (sem necessidade de Box-Cox, já que os modelos escolhidos não exigem distribuição normal das variáveis) | Box-Cox |
