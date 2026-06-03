# Squad 3 — Batch Ecommerce
### Projeto de Engenharia de Dados — Programa de Estagio

---

## O que e esse projeto?


> Pegar os dados brutos das lojas fisicas que estao na nuvem, organiza-los, trata-los e salva-los em um banco de dados estruturado — pronto para ser consultado e analisado.

---

## Arquitetura do Projeto

```
Azure Data Lake (ADLS Gen2)
    Arquivos brutos das lojas
           |
    Databricks (Notebooks)
    Leitura -> Tratamento -> Analise
           |
    Azure SQL Server
    Dados organizados e prontos
```

### O que e cada peca?

| Componente | O que e | Para que serve |
|---|---|---|
| Azure Data Lake (ADLS Gen2) | Um "HD gigante na nuvem" da Microsoft | Armazena os arquivos brutos das lojas |
| Databricks | Uma plataforma de processamento de dados | Onde escrevemos o codigo para tratar os dados |
| Azure SQL Server | Um banco de dados relacional na nuvem | Onde salvamos os dados organizados |
| Service Principal | Uma "conta de servico" com permissoes | Permite que o codigo acesse os dados com seguranca |

---

## Estrutura de Pastas

```
notebooks/
└── squad3/
    └── luiz-portacio/
        ├── README.md                    <- Voce esta aqui
        ├── .env                         <- Credenciais (NUNCA sobe pro GitHub)
        ├── .gitignore                   <- Protege o .env
        ├── config/
        │   └── 00_config                <- Configuracoes e conexoes
        ├── utils/
        │   └── 00_utils                 <- Funcoes reutilizaveis
        ├── ingestion/
        │   ├── feat_squad3_physical_lojas_processamento_batch
        │   └── feat_squad3_physical_itens_venda_caixa_processamento_batch
        └── analysis/
            ├── analysis_squad3_physical_lojas
            └── analysis_squad3_physical_itens_venda_caixa
```

### Por que essa estrutura?

A ideia e separar o projeto em responsabilidades claras:

- config/ — Tudo que e configuracao e conexao fica em um lugar so. Se precisar mudar uma credencial ou um servidor, muda em um unico arquivo.
- utils/ — Funcoes que sao usadas em varios notebooks ficam aqui. Assim, se precisar corrigir uma funcao, corrige em um lugar e todos os notebooks se beneficiam.
- ingestion/ — Notebooks responsaveis por buscar os dados do Data Lake, trata-los e salva-los no banco.
- analysis/ — Notebooks responsaveis por analisar os dados ja salvos no banco e gerar insights.

---

## Bases de Dados Utilizadas

A Squad 3 trabalha com dados das lojas fisicas:

| Arquivo | Tamanho | O que contem |
|---|---|---|
| physical_lojas.csv | 2 KB | Cadastro das 29 lojas fisicas (nome, cidade, estado, CNPJ, peso de vendas) |
| physical_itens_venda_caixa.csv | 277 MB | 2,8 milhoes de registros de itens vendidos no caixa das lojas |

---

## Seguranca — Como protegemos as credenciais

Um dos pontos mais importantes do projeto e nunca expor senhas ou chaves de acesso. Para isso, usamos um arquivo chamado .env que fica apenas na maquina local e nunca e enviado para o GitHub.

```
Como funciona:

.env (arquivo local)          Codigo Python
--------------------          --------------------
client_id=xxxxx       ->      os.getenv("client_id")
jdbc_password=xxxxx   ->      os.getenv("jdbc_password")
```

### Por que isso e importante?

Se as credenciais fossem colocadas diretamente no codigo e enviadas para o GitHub, qualquer pessoa com acesso ao repositorio poderia acessar os dados da empresa. O uso do .env evita esse risco.

---

## Conexoes — Como acessamos os dados

### Conexao com o Data Lake (ADLS Gen2)

Usamos o SDK Azure (azure-storage-file-datalake) para conectar ao Data Lake. Essa e a forma oficial e moderna recomendada pela Microsoft.

```
Por que SDK Azure e nao Mount (dbutils.fs.mount)?

Mount (forma antiga)            SDK Azure (forma moderna)
--------------------            ------------------------
Deprecado                       Recomendado oficialmente
Nao funciona em Serverless      Funciona em qualquer cluster
Requer permissao admin          Funciona com Service Principal
```

### Conexao com o SQL Server

Usamos o formato sqlserver nativo do Databricks para escrever e ler dados. Essa e a forma mais compativel com clusters Serverless.

```
Por que sqlserver nativo e nao JDBC/pyodbc?

pyodbc/JDBC                     sqlserver nativo
--------------------            ------------------------
Requer driver ODBC              Sem instalacao extra
Falha em Serverless             Compativel com Serverless
Configuracao complexa           Simples e direto
```

---

## Como o codigo esta organizado

### 00_config — O painel de controle

E o primeiro notebook a ser executado. Ele:
1. Carrega as credenciais do .env
2. Conecta ao Data Lake
3. Configura a conexao com o SQL Server

Todos os outros notebooks dependem dele.

### 00_utils — A caixa de ferramentas

Contem todas as funcoes reutilizaveis do projeto:

| Funcao | O que faz | Quando usar |
|---|---|---|
| listar_arquivos() | Lista arquivos do Data Lake | Antes de ler qualquer arquivo |
| ler_csv() | Le arquivos CSV pequenos | Arquivos ate 50MB |
| ler_parquet() | Le arquivos Parquet pequenos | Arquivos ate 50MB |
| ler_csv_chunks() | Le arquivos CSV grandes em partes | Arquivos acima de 50MB |
| salvar_tabela() | Salva dados no SQL Server | Arquivos ate 50MB |
| salvar_tabela_lotes() | Salva dados grandes em lotes | Arquivos acima de 50MB |
| consultar_tabela() | Le dados do SQL Server | Para verificacao e analise |
| testar_conexao_sql() | Testa permissoes no banco | Validacao inicial |

### Por que separar em config e utils?

No Databricks, cada notebook e isolado — nao compartilha memoria com outros notebooks. A solucao foi usar o comando %run para importar um notebook dentro de outro:

```
Qualquer notebook de ingestion ou analysis:
    %run ".../utils/00_utils"     <- carrega utils
                |
    00_utils internamente faz:
    %run ".../config/00_config"   <- que carrega o config
```

Assim, com uma unica linha, qualquer notebook tem acesso a todas as configuracoes e funcoes do projeto.

---

## Notebooks de Ingestion — Como os dados sao processados

### O que e ingestion?

Ingestion (ingestao) e o processo de buscar os dados brutos e prepara-los para uso. E como pegar ingredientes crus e prepara-los antes de cozinhar.

### Fluxo de cada notebook de ingestion:

```
1. Listar arquivos disponiveis no Data Lake
2. Ler o arquivo (CSV ou Parquet)
3. Analise Exploratoria
   |- Quantas linhas e colunas?
   |- Quais sao os tipos de dados?
   |- Ha valores nulos?
   └- Ha duplicatas?
4. Tratamentos
   |- Remover duplicatas
   |- Tratar valores nulos
   |- Converter tipos de dados
   |- Padronizar textos
   └- Corrigir valores invalidos
5. Salvar no SQL Server
6. Verificar se os dados foram salvos corretamente
```

### Desafio do arquivo grande (277MB)

O arquivo physical_itens_venda_caixa.csv tem 2,8 milhoes de linhas e 277MB. Carregar tudo de uma vez na memoria causava travamento do sistema.

Solucao adotada:

```
Leitura em chunks               Escrita em lotes
------------------              ------------------
Le 50.000 linhas por vez  ->   Salva 5.000 linhas por vez
Junta tudo no final             Com retry automatico
Nao estoura a memoria           Resiste a falhas de rede
```

### Tratamentos realizados por tabela:

#### physical_lojas (29 linhas)

| Coluna | Problema | Tratamento |
|---|---|---|
| id_loja | Numero inteiro | Convertido para texto |
| cnpj | Numero inteiro (perde zeros) | Convertido para texto com 14 digitos |
| nome_loja | Texto inconsistente | Padronizado para maiusculo |
| cidade_loja | Texto inconsistente | Padronizado para maiusculo |
| estado_loja | Texto inconsistente | Padronizado e validado (siglas UF) |
| peso_vendas | Possiveis negativos | Valor absoluto aplicado |

#### physical_itens_venda_caixa (2,8 milhoes de linhas)

| Coluna | Problema | Tratamento |
|---|---|---|
| id_item_venda | Numero inteiro | Convertido para texto |
| id_transacao | Texto com espacos | Limpeza de espacos |
| codigo_barras_produto | Texto inconsistente | Limpeza e padronizacao |
| quantidade | Float com possiveis negativos | Valor absoluto aplicado |
| preco_unitario_registro | Possiveis negativos | Valor absoluto aplicado |
| valor_total_item | Possiveis negativos | Valor absoluto e validacao de consistencia |

---

## Notebooks de Analysis — O que os dados nos dizem

### O que e analysis?

Analysis (analise) e o processo de extrair insights dos dados ja tratados e salvos no banco. E como olhar para os ingredientes ja preparados e entender o que temos.

### physical_lojas — Insights gerados

- Distribuicao de lojas por estado e cidade
- Ranking de lojas por peso de vendas
- Lojas com maior e menor representatividade

### physical_itens_venda_caixa — Insights gerados

- Receita total das lojas fisicas
- Ticket medio por transacao
- Top 10 produtos por receita
- Top 10 produtos por quantidade vendida
- Analise de transacoes (media de itens, valor medio)

---

## Fluxo Git — Como versionamos o codigo

```
1. Sempre partir da branch dev
   git checkout dev
   git pull origin dev

2. Criar branch seguindo o padrao
   feat/squad3/<tabela>/<descricao>
   ex: feat/squad3/physical_lojas/analise-batch-lojas

3. Desenvolver e commitar
   git add .
   git commit -m "feat: implementa ingestao de physical_lojas"

4. Abrir Pull Request
   feature branch -> dev
   Reviewer: @GeovanyADCancio
```

### Padrao de nomenclatura dos notebooks

```
Ingestion : feat_squad3_<tabela>_processamento_batch
Analysis  : analysis_squad3_<tabela>
```

### Por que esse fluxo?

- Nunca commitar na main ou dev diretamente — evita quebrar o codigo que esta funcionando
- Branches por feature — cada funcionalidade tem seu historico separado
- Pull Request obrigatorio — garante revisao antes de qualquer codigo entrar na branch principal
- Mensagens de commit claras — facilita entender o historico do projeto

---

## Desafios Encontrados e Solucoes

### 1. Cluster Serverless bloqueando configuracoes do Spark

Problema: O cluster Databricks Serverless bloqueia spark.conf.set() e sparkContext.

Solucao: Usar o SDK Azure (azure-storage-file-datalake) diretamente, sem depender do Spark para acessar o Data Lake.

### 2. Mount nao suportado

Problema: dbutils.fs.mount() nao funciona em clusters Serverless.

Solucao: SDK Azure com Service Principal — mais seguro, moderno e sem necessidade de mount.

### 3. Arquivo de 277MB travando

Problema: Carregar 277MB de uma vez estourava a memoria do driver.

Solucao: Leitura em chunks de 50.000 linhas e escrita em lotes de 5.000 linhas com retry automatico.

### 4. Timeout na escrita do SQL Server

Problema: Conexao sendo perdida durante a escrita de grandes volumes de dados.

Solucao: Retry automatico com pausa de 10 segundos entre tentativas e reducao do tamanho dos lotes.

### 5. Driver ODBC nao encontrado

Problema: pyodbc requer instalacao do driver ODBC 18, que nao esta disponivel no Serverless.

Solucao: Usar o conector sqlserver nativo do Databricks, que nao requer instalacao de drivers externos.

### 6. Notebooks nao compartilham contexto

Problema: No Databricks, cada notebook e isolado — variaveis e funcoes de um notebook nao estao disponiveis em outro.

Solucao: Uso do comando %run para carregar o conteudo de um notebook dentro de outro, garantindo que todas as funcoes e configuracoes estejam disponiveis.

---

## Como executar o projeto

### Pre-requisitos

- Acesso ao Databricks com cluster configurado
- Arquivo .env preenchido com as credenciais fornecidas pelo orientador
- Repositorio clonado via Databricks Repos

### Ordem de execucao

```
1. config/00_config
2. utils/00_utils
3. ingestion/feat_squad3_physical_lojas_processamento_batch
4. ingestion/feat_squad3_physical_itens_venda_caixa_processamento_batch
5. analysis/analysis_squad3_physical_lojas
6. analysis/analysis_squad3_physical_itens_venda_caixa
```

Observacao: Os notebooks de ingestion e analysis carregam o config e utils automaticamente via %run — nao e necessario executa-los manualmente antes de cada notebook.

---

## Regras Importantes

- Manipule apenas as tabelas da Squad 3 (squad3.*)
- Nao altere notebooks ou tabelas de outras squads
- O arquivo .env nunca deve ser commitado no GitHub
- Todo acesso ao Data Lake e SQL Server e monitorado
- Em caso de duvida, pergunte antes de alterar qualquer coisa compartilhada

---

## Autor

Luiz Portacio
Squad 3 — Batch Ecommerce
Programa de Estagio — Engenharia de Dados