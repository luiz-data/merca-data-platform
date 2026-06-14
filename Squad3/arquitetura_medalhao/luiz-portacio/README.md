# Squad 3 — Arquitetura Medalhao
## Projeto Batch Lojas Fisicas

---

## O que e esse projeto

Empresa cliente com operacoes de ecommerce e lojas fisicas. A Squad 3 e responsavel pelo processamento batch dos dados das lojas fisicas, implementando uma arquitetura medalhao completa que transforma dados brutos em KPIs de negocio prontos para consumo no Looker e SQL Server.

---

## Arquitetura

```
Azure Data Lake (ADLS Gen2)
raw/batch-data/
   physical_lojas.csv
   physical_vendas_caixa.csv
   physical_itens_venda_caixa.csv
           |
           | leitura via Spark + OAuth
           |
     Bronze (Delta)
     copia fiel + auditoria
           |
           | tratamentos e regras tecnicas
           |
     Silver (Delta)
     dados limpos e enriquecidos
           |
           | agregacoes e KPIs
           |
     Gold (Delta + SQL Server)
     KPIs prontos para o Looker
```

---

## Ambiente Tecnico

| Componente | Detalhe |
|---|---|
| Plataforma | Databricks Serverless |
| Data Lake | Azure Data Lake Storage Gen2 |
| Container leitura | raw/batch-data/ |
| Container escrita | raw/squad3/ |
| Banco de dados | Azure SQL Server |
| Schema SQL | squad3 |
| Autenticacao | Service Principal (OAuth) |
| Formato Delta | Delta Lake via abfss:// |
| Agendamento | Databricks Workflow (toda segunda as 5h) |

---

## Estrutura de Pastas

```
arquitetura_medalhao/
└── luiz-portacio/
    ├── .env                          <- credenciais (nunca subir)
    ├── .gitignore
    ├── README.md
    ├── config/
    │   └── 00_config                 <- variaveis, caminhos, metadados
    ├── utils/
    │   └── 00_utils                  <- funcoes reutilizaveis
    ├── ingestion/
    │   ├── bronze/
    │   │   ├── 01_bronze_physical_lojas
    │   │   ├── 01_bronze_physical_vendas_caixa
    │   │   └── 01_bronze_physical_itens_venda_caixa
    │   ├── silver/
    │   │   ├── 02_silver_physical_lojas
    │   │   └── 02_silver_physical_itens_venda_caixa
    │   └── gold/
    │       ├── 03_gold_kpis_lojas_fisicas
    │       ├── 03_gold_kpi_mom_tipo_pagamento
    │       └── 03_gold_kpis_dados_externos
    └── analysis/
        ├── analysis_physical_lojas
        └── analysis_physical_itens_venda_caixa
```

---

## Tabelas de Entrada

| Arquivo | Tamanho | Linhas | Colunas | Descricao |
|---|---|---|---|---|
| physical_lojas.csv | 2 KB | 29 | 6 | Cadastro das lojas fisicas |
| physical_vendas_caixa.csv | medio | 300.000 | 8 | Transacoes de venda no caixa |
| physical_itens_venda_caixa.csv | 277 MB | 2.841.084 | 6 | Itens vendidos por transacao |

---

## Colunas de Auditoria

| Coluna | Camada | Descricao |
|---|---|---|
| bronze_ingested_at | Bronze | Datetime de ingestao na Bronze (fuso Brasilia) |
| bronze_source_file | Bronze | Caminho real do arquivo de origem (_metadata.file_path) |
| silver_processed_at | Silver | Datetime de processamento na Silver (fuso Brasilia) |
| gold_processed_at | Gold | Datetime de processamento na Gold |
| ano | Bronze/Silver | Ano extraido da data de venda para particao |
| mes | Bronze/Silver | Mes extraido da data de venda para particao |

---

## Camada Bronze

### Regras
- Copia fiel do dado bruto — sem alteracoes
- Todos os campos convertidos para STRING
- Colunas de auditoria adicionadas
- Particionamento por ano e mes (exceto physical_lojas que e dado estatico)
- Validacao de colunas esperadas antes de processar
- Validacao de chave primaria na origem
- Validacao de contagem origem vs destino

### Tabelas Delta

| Tabela | Particao | Descricao |
|---|---|---|
| squad3/bronze/physical_lojas | sem particao | Dado estatico |
| squad3/bronze/physical_vendas_caixa | ano, mes | Particionado por dt_venda |
| squad3/bronze/physical_itens_venda_caixa | ano, mes | Particionado via JOIN vendas |

---

## Camada Silver

### Regras Tecnicas Aplicadas

#### physical_lojas
| Regra | Descricao |
|---|---|
| Regra 1 | id_loja PK — sem nulos, sem duplicatas |
| Regra 2 | cnpj com 14 digitos numericos e unico |
| Regra 3 | estado_loja UF valida (2 letras) |
| Regra 4 | nome_loja "nan" convertido para "Nao Informado" |

#### physical_itens_venda_caixa
| Regra | Descricao |
|---|---|
| Regra 1 | id_item_venda PK — sem nulos, sem duplicatas |
| Regra 2 | id_transacao FK — validado vs physical_vendas_caixa |
| Regra 3 | quantidade > 0 |
| Regra 4 | preco_unitario_registro > 0 |
| Regra 5 | valor_total_item = preco * quantidade (tolerancia R$ 0.05) |

### Enriquecimento Silver
- physical_itens_venda_caixa recebe JOIN com physical_vendas_caixa
- Adiciona: id_loja, dt_venda, tipo_pagamento
- Viabiliza KPIs temporais sem incluir vendas_caixa como tabela principal

---

## Camada Gold

### KPIs Implementados

| Tabela Gold | KPI | Descricao | Particao |
|---|---|---|---|
| gold_kpi_receita_produto_loja_mes | KPI 6 | Receita por produto por loja por mes | ano, mes |
| gold_kpi_top10_produtos_trimestre | KPI 7 | Top 10 produtos por loja por trimestre | ano, trimestre |
| gold_kpi_mom_receita_loja | KPI 8 | Crescimento MoM da receita por loja | ano, mes |
| gold_kpi_ticket_medio_loja | KPI 9 | Ticket medio em itens por loja | ano, mes |
| gold_kpi_receita_loja_ano | KPI 4 | Receita total por loja por ano | ano |
| gold_kpi_crescimento_yoy_loja | KPI 5 | Crescimento YoY por loja | ano |
| gold_kpi_transacoes_loja_mes | KPI 6L | Numero de transacoes por loja por mes | ano, mes |
| gold_kpi_ticket_medio_semestre | KPI 8L | Ticket medio por loja por semestre | ano, semestre |
| gold_kpi_mom_receita_tipo_pagamento | KPI 8ALT | MoM receita por tipo de pagamento | ano, mes |
| gold_kpi_lojas_enriquecimento_ibge | KPI 7L | Lojas enriquecidas com dados IBGE | sem particao |
| gold_kpi_vendas_feriado_loja | KPI 10 | Flag venda em feriado por loja | ano, mes |

### KPIs Pendentes

| KPI | Motivo | Solucao Sugerida |
|---|---|---|
| MoM pereciveis vs secos | Sem coluna de classificacao de produto | Solicitar cadastro de produtos ao orientador |
| Enriquecimento IBGE populacao/renda | API externa implementada sem dados de renda | Solicitar arquivo IBGE com renda ao orientador |

### Dados Externos Utilizados

| Fonte | URL | Dados | Caminho no Data Lake |
|---|---|---|---|
| API IBGE | servicodados.ibge.gov.br | Municipios, UF, regiao | raw/reference-data/ibge_municipios |
| BrasilAPI | brasilapi.com.br/api/feriados/v1/{ano} | Feriados nacionais | raw/reference-data/feriados_nacionais |

---

## Camada Analysis

Os notebooks de analysis consomem exclusivamente as tabelas Gold do SQL Server.
Nao recalculam dados, nao gravam nada — apenas cruzam KPIs para gerar insights.

### analysis_physical_lojas

| Insight | Descricao | KPIs Cruzados |
|---|---|---|
| Insight 1 | Receita vem de volume ou ticket? | KPI 4 x KPI 8L |
| Insight 2 | Crescimento YoY vem de receita ou volume? | KPI 5 x KPI 6L |
| Insight 3 | Peso de vendas vs performance real | KPI 4 x KPI 8L |
| Insight 4 | Concentracao de receita por estado | KPI 4 |
| Insight 5 | Regioes geograficas vs receita (IBGE) | KPI 4 x KPI 7L |

### analysis_physical_itens_venda_caixa

| Insight | Descricao | KPIs Cruzados |
|---|---|---|
| Insight 1 | Produtos ancora — consistentes no top 10 | KPI 7 |
| Insight 2 | MoM receita vs ticket medio | KPI 8 x KPI 9 |
| Insight 3 | Tipo de pagamento vs ticket medio | KPI 8ALT |
| Insight 4 | Homogeneidade de produtos entre lojas | KPI 7 |
| Insight 5 | Impacto de feriados nas vendas | KPI 10 |
| Insight 6 | Quadrante receita alta vs crescimento MoM | KPI 4 x KPI 8 x KPI 6 |

---

## Workflow — squad3_batch_lojas_fisicas_pipeline

### Agendamento
- Frequencia: toda segunda-feira as 5h (fuso America/Sao_Paulo)
- Cron: 0 5 * * 1

### Tasks e Dependencias

| Ordem | Task | Depends on |
|---|---|---|
| 1 | 01_bronze_physical_lojas | nenhuma |
| 2 | 01_bronze_physical_vendas_caixa | 01_bronze_physical_lojas |
| 3 | 01_bronze_physical_itens_venda_caixa | 01_bronze_physical_vendas_caixa |
| 4 | 02_silver_physical_lojas | 01_bronze_physical_lojas |
| 5 | 02_silver_physical_itens_venda_caixa | 01_bronze_physical_itens + 02_silver_physical_lojas |
| 6 | 03_gold_kpis_lojas_fisicas | 02_silver_physical_itens_venda_caixa |
| 7 | 03_gold_kpi_mom_tipo_pagamento | 02_silver_physical_itens_venda_caixa |
| 8 | 03_gold_kpis_dados_externos | 02_silver_physical_itens_venda_caixa (retry x2) |

### Fluxo do Workflow

```
01_bronze_physical_lojas
        |
        +------------------------------+
        |                              |
01_bronze_physical_vendas_caixa   02_silver_physical_lojas
        |                              |
01_bronze_physical_itens______________+
        |
02_silver_physical_itens_venda_caixa
        |
        +------------------+------------------+
        |                  |                  |
03_gold_kpis_         03_gold_mom_       03_gold_kpis_
lojas_fisicas         tipo_pagamento     dados_externos
                                         (retry x2)
```

---

## Decisoes Tecnicas

### Por que SDK Azure para leitura de CSV?
O cluster Serverless bloqueia spark.conf.set() e sparkContext.
A autenticacao OAuth e passada via .options(**adls_options) diretamente
no spark.read, sem necessidade de configuracao global.

### Por que saveAsTable nao foi utilizado?
O projeto grava fisicamente no ADLS via abfss:// usando write_delta().
saveAsTable usa o Unity Catalog gerenciado e nao grava no caminho
fisico desejado (squad3/bronze, silver, gold).

### Por que physical_vendas_caixa e tabela de referencia e nao principal?
O escopo da Squad 3 e physical_lojas e physical_itens_venda_caixa.
physical_vendas_caixa e usada apenas como JOIN para obter dt_venda
e id_loja, viabilizando KPIs temporais sem ser tabela principal.

### Por que current_timestamp() em vez de datetime.now()?
current_timestamp() e uma funcao nativa do Spark que usa o
timestamp do cluster de forma distribuida. datetime.now() usa
o timestamp do driver Python, menos preciso em processamentos
distribuidos.

### Por que _metadata.file_path para bronze_source_file?
_metadata.file_path e um metadado nativo do Spark que retorna
o caminho real e dinamico do arquivo lido. Uma string estatica
como lit("physical_lojas.csv") nao reflete o caminho completo
e real do arquivo no Data Lake.

### Por que MoM por tipo de pagamento substituiu MoM pereciveis vs secos?
O arquivo physical_itens_venda_caixa nao possui coluna de
classificacao de produto (perecivel/seco). O KPI alternativo
usa tipo_pagamento que esta disponivel via JOIN com vendas_caixa,
sendo igualmente relevante para decisoes de infraestrutura.

---

## Seguranca

- Credenciais armazenadas exclusivamente no arquivo .env
- .env nunca commitado no GitHub (.gitignore configurado)
- Autenticacao via Service Principal (client_id, tenant_id, client_secret)
- Todo acesso ao Data Lake e SQL Server e monitorado
- Manipulacao exclusiva das tabelas squad3.*

---

## Variaveis de Ambiente (.env)

```
# Azure Data Lake (ADLS GEN2)
client_id=
tenant_id=
client_secret=
storage_account_name=
container_name=raw
batch_data_path=batch-data
bronze_path=squad3/bronze
silver_path=squad3/silver
gold_path=squad3/gold

# SQL Server (Azure)
jdbc_hostname=
jdbc_database=
jdbc_username=
jdbc_password=
sql_schema=squad3
```

---

## Fluxo Git

```
Branch: feat/squad3/physical_lojas/medallion-architecture
Base  : dev
Reviewer: @GeovanyADCancio

Padrao de commit:
feat: descricao clara do que foi implementado

Nunca commitar em main ou dev diretamente
PR sempre aponta para dev
```

---

## Dicionario de Dados Gold

### Granularidade das tabelas Gold

| Tabela | Granularidade |
|---|---|
| gold_kpi_receita_produto_loja_mes | 1 linha por produto por loja por mes |
| gold_kpi_top10_produtos_trimestre | 1 linha por produto (top 10) por loja por trimestre |
| gold_kpi_mom_receita_loja | 1 linha por loja por mes |
| gold_kpi_ticket_medio_loja | 1 linha por loja por mes |
| gold_kpi_receita_loja_ano | 1 linha por loja por ano |
| gold_kpi_crescimento_yoy_loja | 1 linha por loja por ano |
| gold_kpi_transacoes_loja_mes | 1 linha por loja por mes |
| gold_kpi_ticket_medio_semestre | 1 linha por loja por semestre |
| gold_kpi_mom_receita_tipo_pagamento | 1 linha por loja por tipo pagamento por mes |
| gold_kpi_lojas_enriquecimento_ibge | 1 linha por loja |
| gold_kpi_vendas_feriado_loja | 1 linha por loja por feriado por mes |

---

## Autor

Luiz Portacio
Squad 3 — Batch Ecommerce