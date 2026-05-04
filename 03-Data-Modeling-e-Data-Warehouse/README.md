# 03 — Data Modeling e Data Warehouse no Amazon Redshift

> [!NOTE]
> Este README é a **proposta pedagógica** dos laboratórios da Aula 3. Ele descreve o que será implementado, por que é possível dentro das restrições do AWS Academy Learner Lab e quais são os passos macros. A implementação dos exercícios virá em subpastas próprias (`01-provisionamento`, `02-modelagem-e-carga`, `03-analise-dimensional`).

---

## Contexto da aula

A Aula 3 fecha a disciplina deslocando o foco de "onde os dados ficam" (storage, open table formats) para **"como os dados passam a significar algo para o negócio"**. O material conceitual atravessa:

1. Separação entre sistemas operacionais e analíticos
2. **Grain** como contrato de engenharia e negócio
3. Fatos, dimensões, star schema
4. Tipos de fato (transacional, snapshot periódico, snapshot acumulado) e comportamento das medidas
5. Surrogate keys, **SCD Tipo 1 e Tipo 2**
6. Dimensões conformadas e bus matrix
7. Lake x warehouse x lakehouse como composição
8. Amazon Redshift: arquitetura MPP, colunar, RA3, Serverless
9. Distkey, sort key, COPY, Spectrum, data sharing, materialized views

O laboratório precisa permitir ao aluno **executar** essa cadeia inteira em um caso real, não apenas ler sobre ela. E precisa caber nas restrições do Learner Lab.

---

## Restrições do AWS Academy Learner Lab que moldam o desenho

Antes de desenhar qualquer exercício, é preciso respeitar as limitações do ambiente. As decisões arquiteturais abaixo são **consequência direta** dessas restrições:

| Restrição do Learner Lab | Impacto no desenho |
|--------------------------|--------------------|
| **Não é possível criar IAM roles** | Usaremos exclusivamente a role pré-criada `LabRole` (ARN: `arn:aws:iam::<ACCOUNT_ID>:role/LabRole`) para Redshift, Glue e S3. Nada de custom roles. |
| **Região limitada a `us-east-1` e `us-west-2`** | Tudo na `us-east-1` (padrão da disciplina). |
| **Redshift**: tipo `ra3.large`, máximo **2 nós** provisionados (Serverless **não listado** no PDF → indisponível) | Usaremos **Redshift provisionado**, cluster single-node `ra3.large`. Sem Serverless, sem S3 Tables, sem Lake Formation, sem DataZone (não listados no PDF). |
| **EC2**: apenas `nano`, `micro`, `small`, `medium`, `large`; máximo 9 instâncias concorrentes | Nenhum lab exige EC2 direto. Tudo roda via Codespaces + serviços gerenciados. |
| **Budget consumível** (créditos limitados) | Recursos pagos por demanda (Serverless, S3, Glue). Scripts de **teardown** obrigatórios. |
| **Sessão de 4h** — recursos persistem entre sessões, mas compute pode ser parada | Laboratório orientado a retomada; dados ficam no S3 e no namespace Redshift. |
| **Glue ETL**: workers `G.1X` ou `Standard`, máximo 10 workers, concorrência 1 | Se precisarmos converter dados, Glue Job pequeno resolve. Alternativa preferida: conversão no próprio Codespaces com Python/pandas. |
| **Sem `create IAM role`** em serviços → alguns serviços falham | Sempre passar `LabRole` explicitamente no CloudFormation/Terraform. |

> [!IMPORTANT]
> O aluno **não vai provisionar infraestrutura na mão**. Todo o cluster/namespace Redshift, bucket S3 e catálogo Glue serão criados via **Terraform** (IaC) na pasta `01-provisionamento/`. O foco da aula é **modelar e consultar**, não clicar em console.

---

## Dataset escolhido — TPC-H em Parquet (1-2 GB)

Para garantir que **todos os alunos executem a mesma query e obtenham o mesmo resultado**, o laboratório usará um dataset público, determinístico e pronto:

### TPC-H Scale Factor 1 (~1.2 GB em Parquet)

- **Fonte primária (bucket público AWS)**: [`s3://redshift-downloads/TPC-H/2.18/1GB/`](https://redshift-downloads.s3.amazonaws.com/TPC-H/2.18/1GB/) — bucket público mantido pela AWS, usado em tutoriais oficiais do Redshift. Contém os 8 arquivos do TPC-H SF1 em formato texto delimitado (`|`).
- **Conversão para Parquet**: feita uma única vez pelo [`scripts/load_tpch.sh`](01-provisionamento/scripts/load_tpch.sh) dentro do Codespaces (download → convert → upload no bucket do aluno). Script usa `pandas` + `pyarrow` (já disponíveis no devcontainer).
- **Alternativa 100% pronta em Parquet** (caso o passo de conversão seja indesejado): [`s3://blogpost-sparkoneks-us-east-1/blog/BLOG_TPCDS-TPCH-PIGTPCH/tpch/`](https://registry.opendata.aws/) — outros mirrors AWS de TPC-H em Parquet estão catalogados no [AWS Open Data Registry](https://registry.opendata.aws/).
- **Especificação oficial do benchmark**: [TPC-H Benchmark — TPC.org](https://www.tpc.org/tpch/)
- **Repositório de referência AWS**: [awslabs/amazon-redshift-utils](https://github.com/awslabs/amazon-redshift-utils)
- **Formato final no laboratório**: Parquet (snappy-compressed) no bucket `dw-lab-<ACCOUNT_ID>/raw/tpch/`
- **Tamanho**: ~1.2 GB distribuído em **8 tabelas relacionais**
- **Domínio**: sistema de compras / supply chain (cenário clássico, rico em relacionamentos)

### Tabelas do dataset (fonte operacional simulada)

| Tabela | Descrição | Linhas aprox. (SF1) |
|--------|-----------|---------------------|
| `lineitem` | Itens de pedido (maior tabela, ~600 MB) | 6.000.000 |
| `orders` | Pedidos | 1.500.000 |
| `customer` | Clientes | 150.000 |
| `part` | Produtos/peças | 200.000 |
| `supplier` | Fornecedores | 10.000 |
| `partsupp` | Relação produto-fornecedor | 800.000 |
| `nation` | Países | 25 |
| `region` | Regiões | 5 |

### Por que TPC-H e não gerar do zero?

- **Determinismo**: mesmas queries produzem os mesmos números em toda a turma (requisito explícito do professor).
- **Realismo**: dataset clássico da indústria, usado em benchmarks e documentação AWS.
- **Volume adequado**: ~1.2 GB cabe nas restrições e é suficiente para observar efeitos de distkey, sort key e Spectrum.
- **Rico em dimensões**: 8 tabelas permitem modelagem fato/dimensão, dimensões conformadas e cenários de SCD.
- **Está disponível no script da Aula 2** (TPC-DS). TPC-H é irmão dele — preferimos TPC-H porque o modelo é mais enxuto para uma aula única.

> [!NOTE]
> Alternativa considerada: TPC-DS (usado no lab `02`). Descartado porque tem 24 tabelas — excesso para o escopo da aula. TPC-H entrega o mesmo valor pedagógico com modelo mais legível.

---

## Proposta: dois exercícios conectados

A aula será dividida em **dois laboratórios sequenciais** que compartilham dataset, infraestrutura e modelo. Um depende do outro — o aluno monta a casa e depois mora nela.

> [!IMPORTANT]
> **A tese central dos dois labs:** a mesma pergunta de negócio pode produzir respostas diferentes quando a modelagem muda, e a modelagem muda porque o negócio evolui. O aluno sai da aula entendendo que **"o número está certo" ou "o número está errado" não é uma pergunta técnica — é uma pergunta de contrato semântico**. Essa é a distinção entre um warehouse confiável e uma coleção de tabelas que "por acaso" somam os mesmos valores hoje.

### O que o Redshift permite fazer — e por que isso importa para modelagem

Antes dos exercícios, vale fixar **as capacidades do Redshift que habilitam os cenários pedagógicos** deste lab. Cada uma delas aparece em pelo menos um dos dois exercícios abaixo, provocando trade-offs concretos:

| Capacidade do Redshift | O que permite na prática | Como aparece no lab |
|------------------------|--------------------------|---------------------|
| **Colunar + MPP por slices** | Scans seletivos de poucas colunas em tabelas grandes; paralelismo por slice mesmo em single-node | `EXPLAIN` mostrando `scan` colunar; comparação de tempo entre `SELECT *` vs. `SELECT col` |
| **`DISTKEY` / `DISTSTYLE ALL/AUTO/EVEN/KEY`** | Controla como linhas são espalhadas entre slices → define se joins são locais ou exigem redistribuição | Recriar `f_vendas` com 3 distkeys diferentes (customer, data, even) e medir `EXPLAIN ANALYZE` |
| **`SORTKEY` compound/interleaved** | Ordena dados fisicamente, habilita zone maps e block skipping em filtros por faixa | Consulta de 1 mês vs. 1 ano com e sem sort key por `data_sk` |
| **`COPY FROM 's3://...'`** | Ingestão paralela de Parquet/CSV com `IAM_ROLE 'LabRole ARN'` | Carga inicial das dimensões e fato; split de arquivos influencia paralelismo |
| **`CREATE TABLE AS SELECT` (CTAS)** | Materializa resultado de query em tabela física otimizável | Construção das dimensões a partir do modelo operacional |
| **`MERGE INTO` (e alternativa `INSERT+UPDATE`)** | Aplicação idempotente de SCD Tipo 2, versionamento de dimensão | Evolução da `dim_customer` com mudança de segmento preservando histórico |
| **Materialized Views** com `AUTO REFRESH` | Pré-computa agregações caras, refresh gerenciado pelo serviço | Dashboard de receita mensal por região (consumo recorrente) |
| **Views regulares** | Camada semântica versionável, apelidos de negócio | `v_receita_liquida` encapsulando a fórmula `extendedprice * (1 - discount) * (1 + tax)` |
| **Compressão colunar (`ENCODE AUTO`)** | Menos I/O, armazenamento enxuto | Comparação de tamanho físico da mesma tabela com encoding manual ruim vs. `AUTO` |
| **Workload Management (WLM)** / `STL_QUERY` | Observabilidade de queries, tempo de fila, tempo de exec | Leitura das system tables para debugar uma query lenta |
| **Data type `SUPER` + PartiQL** | Modelagem de dados semiestruturados sem normalizar antes da carga | Extensão opcional: carga de atributos JSON de pedido em coluna SUPER |
| **Redshift ML (`CREATE MODEL`)** | Treino e inferência via SQL, usando SageMaker por baixo | Menção em discussão — não executado por custo, mas contextualizado |

O ponto aqui não é cobrir todos os recursos. É mostrar que **cada recurso existe como resposta a uma dor de negócio específica** — e que usar o recurso errado (ou não usar) na hora errada muda o resultado da query ou seu custo.

---

### A narrativa dos dois labs — "a mesma empresa, três momentos"

Para conectar os dois exercícios, adotamos uma **empresa fictícia** que evolui ao longo do lab. É o mesmo dataset TPC-H, mas reinterpretado como uma narrativa de maturidade analítica:

- **Momento 1** — *"Recém-saímos do OLTP"* — a empresa acabou de colocar um Redshift, carregou uma cópia fiel das tabelas operacionais (8 tabelas TPC-H relacionais) e começou a rodar relatórios direto nelas. Funciona, mas dói.
- **Momento 2** — *"Modelamos para o negócio"* — o time de dados desenha um star schema, declara o grain, cria dimensões desnormalizadas com surrogate keys, SCD2 para atributos que importam historicamente, e carrega a fato. As mesmas perguntas ficam mais simples, mas **algumas respostas mudam**.
- **Momento 3** — *"O negócio mudou"* — uma regra de negócio nova entra em vigor (exemplo: reclassificação de segmentos de cliente, novo imposto regional, fusão de canais de venda). A mesma pergunta feita há 3 meses agora dá outro número. Por quê? E o que isso diz sobre o modelo?

Essa narrativa não é decorativa. Ela **força o aluno a enfrentar a ambiguidade semântica na prática**. É aqui que modelagem dimensional deixa de ser "diagrama bonito" e vira contrato de significado.

---

### Lab 03.1 — Do OLTP ao Star Schema: três modelagens, três respostas

**Duração estimada**: 75-90 min

**Objetivo pedagógico**: levar o aluno a rodar **exatamente a mesma pergunta de negócio** em três modelagens diferentes da mesma base, observar que os números divergem e entender **por que cada divergência tem uma justificativa legítima**. No final, o aluno escolhe qual modelagem vai servir o dashboard e precisa defender a escolha.

**A pergunta-âncora do lab:**

> *"Qual foi a receita líquida total do segmento `AUTOMOBILE` no ano de 1995, por região do cliente?"*

Essa pergunta foi escolhida porque:
- Usa **4 tabelas distintas** do TPC-H (`lineitem`, `orders`, `customer`, `nation+region`), obrigando join — expõe impacto de distkey.
- Envolve **cálculo composto** (`receita líquida = extendedprice × (1 - discount)` — essa fórmula do próprio benchmark TPC-H vira contrato).
- Tem **dimensão temporal** (ano de 1995), expondo impacto de sort key.
- Tem **filtro categórico** (segmento), expondo impacto de SCD — segmento pode mudar ao longo do tempo!
- Tem **agrupamento hierárquico** (região), expondo impacto de desnormalização.

**Três modelagens que o aluno implementa:**

#### Modelagem A — "Espelho do OLTP" (schema `oltp_mirror`)

Carrega as 8 tabelas TPC-H como são, com as mesmas PKs/FKs naturais. Nenhuma dimensão, nenhuma fato, nenhuma surrogate key.

- Simples de carregar (COPY direto)
- Consultas repetem joins longos e conhecimento do modelo operacional
- Filtros dependem de colunas espalhadas (`o_orderdate`, `c_mktsegment`, `r_name`)
- **Sem histórico**: se a origem reclassificar um cliente de `AUTOMOBILE` para `BUILDING`, a pergunta de 1995 passa a retornar **um número diferente** na próxima carga — e ninguém percebe.

#### Modelagem B — "Star schema clássico" (schema `dw_star`)

Grain declarado: **uma linha de `f_vendas` = um item de pedido vendido**.

- `dim_data`, `dim_customer` (SCD Tipo 1), `dim_produto`, `dim_supplier`, `dim_geografia` (achata `nation + region`), `f_vendas`
- Surrogate keys geradas no warehouse (`customer_sk`, etc.)
- Distkey em `customer_sk`, sortkey em `data_sk`
- Cálculo de receita líquida **materializado** na fato como coluna `vr_receita_liquida` — contrato explícito
- A consulta fica 1 join + 1 filtro. Simples, rápida.
- **Mas**: SCD Tipo 1 sobrescreve segmento do cliente → se o cliente trocou de segmento depois da venda, a resposta de 1995 reflete o **segmento atual**, não o da época. Isso **pode ou não ser o que o negócio quer**.

#### Modelagem C — "Star schema com histórico" (schema `dw_star_scd2`)

Mesmo modelo do B, mas `dim_customer` agora é **SCD Tipo 2**.

- Colunas `valid_from`, `valid_to`, `is_current`, `customer_sk` versionada
- A fato aponta para a versão **vigente no momento da venda**
- A mesma query agora retorna um terceiro número: a **receita atribuída ao segmento que o cliente tinha naquela venda**

**O momento de virada do lab:**

O aluno roda a query-âncora nos três schemas e obtém **três números diferentes**. Exemplo ilustrativo:

| Modelagem | Receita AUTOMOBILE em 1995 (AMERICA) | Por quê |
|-----------|---------------------------------------|---------|
| A (OLTP mirror) | $X₁ | Reflete segmento atual do cliente (dado sobrescrito pela origem) |
| B (Star SCD1) | $X₂ | Igual a A conceitualmente, mas pode divergir por diferenças de cálculo de receita líquida |
| C (Star SCD2) | $X₃ | Reflete segmento que o cliente tinha **em 1995** |

> [!NOTE]
> Para o aluno conseguir reproduzir isso de forma didática, o `load_tpch.sh` **injeta de propósito** uma evolução: ~5% dos clientes da TPC-H recebem reclassificação de segmento com data de alteração posterior a 1995. Essa alteração é registrada em uma tabela `customer_history` carregada junto. A tabela `customer` original do TPC-H vira o "snapshot atual" e a `customer_history` alimenta o SCD2.

**Conceitos exercitados (expandidos)**:
- Grain declaration explícito e **comparação com o não-grain** do espelho OLTP
- Diferença entre modelo operacional e analítico (lado a lado no mesmo cluster)
- Surrogate keys (`customer_sk` vs. `c_custkey`)
- **SCD Tipo 1 vs. Tipo 2** com consequências numéricas visíveis
- Medidas aditivas (`l_quantity`), semi-aditivas (estoque, se extendermos) e **derivadas** (receita líquida)
- Materialização vs. cálculo on-the-fly (receita como coluna vs. expressão em view)
- `COPY` com `IAM_ROLE 'LabRole'`, split de arquivos, `MAXERROR`
- `DISTSTYLE` / `DISTKEY` / `SORTKEY` com consequências observáveis em `EXPLAIN ANALYZE`
- `MERGE INTO` para manter dimensão SCD2 (idempotência de carga)

**Fluxo macro**:

1. `terraform apply` em `01-provisionamento/` → sobe tudo
2. `bash scripts/load_tpch.sh` → baixa TPC-H, injeta `customer_history`, envia Parquet para S3
3. Aluno abre Redshift Query Editor v2
4. **Etapa A**: cria schema `oltp_mirror`, `COPY` das 8 tabelas, roda a query-âncora. Anota o número.
5. **Etapa B**: cria schema `dw_star`, cria `dim_*` e `f_vendas` com CTAS, roda a query-âncora. Anota o número. Compara.
6. **Etapa C**: cria schema `dw_star_scd2`, reconstrói `dim_customer` com SCD2 usando a `customer_history`, recarrega a fato apontando para a versão correta. Roda a query-âncora. Anota. Compara os três.
7. **Discussão guiada** (no README do lab): "qual dos três números é o certo?"
   - Resposta: **depende da pergunta exata do negócio**. Queremos "receita do segmento como está hoje" (A/B), "receita do segmento que o cliente tinha na época da venda" (C), ou algo entre?
8. O aluno **escolhe** uma das três modelagens e **documenta a escolha** em um `DECISION.md` (ativo real de engenharia).
9. Teardown opcional ao final (`terraform destroy`).

---

### Lab 03.2 — Evolução do negócio: quando a modelagem tem que mudar

**Duração estimada**: 60-75 min

**Objetivo pedagógico**: simular que **o negócio mudou** e observar o que isso faz com o modelo escolhido. O aluno vê que evolução de negócio é uma força **constante** sobre o warehouse, e que decisões de modelagem tomadas no dia 1 raramente sobrevivem ao ano 2 sem ajuste. O recurso do Redshift vira consequência do problema de negócio, não o contrário.

**Três evoluções aplicadas sobre o star schema do Lab 03.1:**

#### Evolução 1 — "Abrimos um novo canal e queremos medir margem real"

A empresa agora vende também em marketplaces com comissão variável. A receita líquida deixa de ser `price × (1 - discount)` e passa a ser `price × (1 - discount) × (1 - commission_rate)`. O campo `commission_rate` não existe no TPC-H — vamos simular adicionando um atributo à dimensão `dim_supplier` (cada fornecedor tem sua comissão).

- **Dor**: todas as views e MVs que calculavam receita ficam desatualizadas
- **Resposta técnica no Redshift**: criar `v_receita_liquida_v2` (nova view), preservar `v_receita_liquida` (v1) para histórico, criar Materialized View para a v2 e comparar performance
- **Decisão pedagógica**: recalcular histórico ou só aplicar para vendas novas? Como isso conversa com SCD?
- **Recursos do Redshift envolvidos**: Materialized Views com `AUTO REFRESH`, Views encadeadas, `EXPLAIN` comparando MV × view pura

#### Evolução 2 — "Redefinimos o conceito de 'cliente ativo'"

Antes, cliente ativo era "fez ao menos uma compra". Agora, é "fez ao menos uma compra nos últimos 12 meses **e** tem saldo devedor < 5000". A dimensão `dim_customer` precisa ganhar atributos calculados. Aparece a tensão: **atributo de dimensão ou fato snapshot periódico?**

- **Dor**: a pergunta "quantos clientes ativos temos?" muda de definição **e** muda ao longo do tempo (cliente vira ativo e inativo ciclicamente)
- **Resposta técnica no Redshift**: introduzir uma **fato snapshot periódico** `f_customer_status_mensal` (uma linha por cliente × mês) com flag `is_active`, em paralelo à `dim_customer`
- **Decisão pedagógica**: isso vira coluna na dimensão (SCD Tipo 2) ou uma fato própria? Cada escolha tem consequência — a fato snapshot permite contagens agregadas rápidas, a dimensão SCD2 preserva contexto histórico nas queries que já existem.
- **Recursos do Redshift envolvidos**: segundo tipo de fato (snapshot), `DISTKEY` por `customer_sk` (para colocalizar com `dim_customer`), `SORTKEY` por `data_sk`, window functions sobre o snapshot

#### Evolução 3 — "Comprometemo-nos com um SLA de 5s no dashboard executivo"

O dashboard de receita por região × mês × segmento tem que carregar em menos de 5s, mesmo com dados crescendo 3x ao ano. A query atual faz hash join + redistribute porque a distkey foi colocada em `customer_sk`, mas o filtro mais frequente é por `data_sk`.

- **Dor**: a distkey que era ótima para o Lab 03.1 passou a ser um problema
- **Resposta técnica no Redshift**: reconstruir `f_vendas` com `DISTKEY(data_sk)` ou `DISTSTYLE AUTO`, comparar `EXPLAIN ANALYZE` antes e depois, criar Materialized View pré-agregada para o dashboard
- **Decisão pedagógica**: mudar distkey é custoso (recria tabela). Vale? Ou criar uma **segunda** cópia pré-agregada (uma MV) é melhor?
- **Recursos do Redshift envolvidos**: `DISTSTYLE AUTO`, Materialized View com refresh agendado, `STL_QUERY`/`SYS_QUERY_HISTORY` para medir latência real

**Conceitos exercitados (expandidos)**:
- Queries analíticas clássicas (top-N, drill-down, YoY) como **aplicação**, não como objetivo
- `EXPLAIN` e `EXPLAIN ANALYZE` usados para **justificar** decisão de redesign
- Impacto real de `DISTKEY` e `SORTKEY` medido, não postulado
- Materialized Views como **resposta a SLA**, não como feature decorativa
- Views versionadas como **camada semântica contratual**
- Tipos de fato: transacional vs. snapshot periódico como **escolha arquitetural** derivada de pergunta de negócio
- Observabilidade via `SVV_TABLE_INFO`, `STL_QUERY`, `SYS_*` views
- Discussão: **quando recalcular histórico vs. quando versionar cálculo**

**Fluxo macro**:

1. Aluno parte do schema escolhido no final de 03.1 (uma das três modelagens)
2. **Evolução 1**: implementa nova fórmula de receita, avalia MV vs. view, compara números históricos
3. **Evolução 2**: modela `is_active` como dimensão SCD2 **e** como fato snapshot, compara ambas em queries reais
4. **Evolução 3**: roda a query do "dashboard executivo" na modelagem atual, mede tempo, muda distkey/cria MV, remede, documenta ganho
5. **Reflexão final** (guiada por perguntas no README do lab):
   - *"Você recalcularia o histórico com a nova fórmula ou manteria o número antigo congelado?"*
   - *"O que é 'correto': o número da métrica no dia em que foi publicada, ou o número recalculado com a regra atual?"*
   - *"Se o CEO pedir 'a mesma tabela de 2023', você entrega a tabela como ela era ou a tabela como ela fica aplicando as regras de hoje aos dados antigos?"*

> [!TIP]
> Essas perguntas não têm resposta única. O objetivo é que o aluno saia com **vocabulário para negociar com stakeholders**, não com um algoritmo pronto.

---

### Por que essa estrutura é pedagogicamente diferente

A maioria dos labs de Redshift que existem por aí é "aqui está o dataset, aqui estão as queries, olhe o `EXPLAIN`". Isso ensina sintaxe, não engenharia.

Os dois labs aqui são estruturados para ensinar três coisas que **só aparecem quando há fricção entre modelagens alternativas**:

1. **Contrato semântico importa mais que sintaxe SQL** — o aluno vê a mesma query produzindo três números legítimos e precisa escolher.
2. **Modelagem é uma conversa contínua com o negócio, não um entregável único** — as evoluções do 03.2 são inevitáveis na vida real.
3. **Recursos do Redshift (MV, distkey, sortkey, SCD) são respostas a perguntas de negócio específicas** — não são "boas práticas" abstratas. O aluno usa cada recurso quando o problema do dia pede.

---

## Infraestrutura a ser provisionada

Toda a infraestrutura sobe via **Terraform** na pasta `01-provisionamento/`. Nenhum passo manual no console exceto copiar credenciais do AWS Academy.

### Recursos criados pelo Terraform

```
┌─────────────────────────────────────────────────────────┐
│  S3 Bucket: dw-lab-<ACCOUNT_ID>                          │
│    ├── raw/tpch/         (Parquet original)              │
│    ├── staging/          (áreas de trabalho)             │
│    └── results/          (output de unload / exports)    │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  Glue Data Catalog                                       │
│    └── database: tpch_raw                                │
│         ├── table: lineitem  (external, aponta p/ S3)   │
│         ├── table: orders                                │
│         ├── table: customer                              │
│         └── ... (8 tabelas)                              │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  Amazon Redshift (provisionado, single-node)             │
│    ├── cluster identifier: dw-aula3-<ACCOUNT_ID>         │
│    ├── node type: ra3.large                              │
│    ├── node count: 1                                     │
│    ├── IAM role associada: LabRole (pré-existente)       │
│    ├── DB name: dw_mba                                   │
│    ├── master user: admin                                │
│    ├── publicly accessible: true (pra Codespaces chegar) │
│    └── schemas criados nos labs:                         │
│          ├── oltp_mirror     (Lab 03.1 — Modelagem A)    │
│          ├── dw_star         (Lab 03.1 — Modelagem B)    │
│          ├── dw_star_scd2    (Lab 03.1 — Modelagem C)    │
│          └── dw_evolucao     (Lab 03.2 — extensões)      │
└─────────────────────────────────────────────────────────┘
```

### Por que Redshift provisionado single-node `ra3.large`?

1. **É o que o Learner Lab permite** — Serverless não aparece no PDF de restrições, portanto tratamos como indisponível.
2. **Single-node cabe no budget** — 1 × `ra3.large` é o menor footprint pagante do Redshift; suficiente para o volume do TPC-H SF1.
3. **Pedagogicamente rico mesmo com 1 nó** — mesmo sem múltiplos nós, o aluno observa distribuição por slices, sort keys, compressão colunar, materialized views e MPP interno por slice. Single-node `ra3.large` tem 2 slices, suficiente para ver redistribuição de dados em `EXPLAIN ANALYZE`.
4. **Pausa suportada** — `aws redshift pause-cluster` reduz custo entre aulas (aluno pausa ao final da sessão).
5. **Aceita a role `LabRole`** associada via IAM role attachment — sem criar role nova.

> [!WARNING]
> O cluster **continua consumindo budget mesmo ocioso** enquanto não for pausado ou deletado. O `terraform destroy` no final da aula é parte obrigatória do fluxo.

### Por que Terraform e não CloudFormation ou shell?

1. **Já instalado no devcontainer** (nenhuma novidade para o aluno).
2. **State explícito** permite `terraform destroy` confiável ao final da aula.
3. **Mais legível** que CloudFormation para alunos que vão revisar depois.
4. **Alinhado com a indústria** — disciplina também serve para formar engenheiros.

### Shell script será usado quando?

Apenas para a **preparação do dataset** (uma única vez):

```
.devcontainer → Codespaces pronto
         │
         ▼
01-provisionamento/
    ├── main.tf          → terraform apply
    ├── variables.tf
    ├── outputs.tf
    └── scripts/
         └── load_tpch.sh → baixa TPC-H, envia para S3, cria partições no Glue
```

O shell script `load_tpch.sh` é o único script — e só roda porque:
- Gerar TPC-H puramente em Terraform é inviável (é download + upload de dados).
- Fazer em Python puro funciona, mas bash + AWS CLI é mais direto para este tipo de tarefa.
- Ele roda **depois** do Terraform (o bucket precisa existir).

---

## Estrutura planejada da pasta

```
03-Data-Modeling-e-Data-Warehouse/
├── README.md                          ← este arquivo (proposta)
├── 01-provisionamento/
│   ├── README.md                      ← como subir o ambiente
│   ├── main.tf                        ← Redshift provisionado + S3 + Glue DB
│   ├── variables.tf
│   ├── outputs.tf                     ← expõe endpoint Redshift, bucket, etc
│   ├── versions.tf
│   └── scripts/
│       └── load_tpch.sh               ← dataset ingestion (uma vez)
├── 02-modelagem-e-carga/                ← Lab 03.1 (três modelagens, três respostas)
│   ├── README.md                       ← passo a passo narrativo + todos os SQLs inline
│   ├── DECISION_TEMPLATE.md            ← template p/ aluno documentar escolha
│   └── img/
│       ├── arquitetura-03-1.drawio
│       └── arquitetura-03-1.png
└── 03-analise-dimensional/              ← Lab 03.2 (três evoluções do negócio)
    ├── README.md                       ← passo a passo narrativo + todos os SQLs inline
    └── img/
        ├── arquitetura-03-2.drawio
        └── arquitetura-03-2.png
```

> [!NOTE]
> Os SQLs ficam **inline** nos READMEs, para o aluno copiar e colar no Redshift Query Editor v2. Esse é o mesmo padrão dos Labs 01 (Storage) e 02 (Open Table Format).

---

## Como o aluno acessa o Redshift (sem EC2)

O Learner Lab tem restrição de EC2, mas **isso não é um problema** para este lab. Caminhos disponíveis, em ordem de preferência:

| Caminho | Onde roda | Quando usar |
|---------|-----------|-------------|
| **Redshift Query Editor v2** (console AWS) | Browser | ✅ Padrão da disciplina. SQL direto, output visual, zero setup. |
| **psql no Codespaces** via endpoint público | Codespaces | Para scripts batch e integração com o repositório. |
| **boto3 + redshift-data API** (Python) | Codespaces | Opcional — quem quer automação. |

O Codespaces já tem AWS CLI, Python e psql disponíveis (ver [.devcontainer/README.md](../.devcontainer/README.md)). O único ajuste necessário é abrir o endpoint Redshift para o IP de saída do Codespaces — o Terraform já configura `publicly_accessible = true` com security group permissivo **apenas para o laboratório**.

> [!WARNING]
> `publicly_accessible` está OK no contexto de laboratório educacional com dataset público TPC-H. Em produção, nunca faria sentido — deveria usar VPC peering, PrivateLink ou Redshift Data API.

---

## Por que isto é possível no Learner Lab — validação ponto a ponto

| Serviço necessário | Permitido? | Observação |
|--------------------|-----------|------------|
| **Amazon S3** | ✅ Sim | LabRole tem acesso |
| **Amazon Redshift (provisionado)** | ✅ Sim | `ra3.large`, single-node (limite do PDF: máx 2 nós) |
| **AWS Glue** (Data Catalog) | ✅ Sim | Assume LabRole. Usado apenas para catalogar o TPC-H em S3 (referência para `COPY FROM`) |
| **Redshift Spectrum** | ⚠️ Não listado no PDF | **Não usamos**. Toda consulta é contra tabelas internas do Redshift carregadas via `COPY` |
| **CloudFormation / Terraform** | ✅ Sim | Ambos permitidos. Terraform via CLI no Codespaces |
| **IAM LabRole** | ✅ Pré-existente | Passado como parâmetro em todo recurso |
| **EC2** | ❌ Não usaremos | Codespaces substitui — aluno não precisa de instância |
| **VPC default** | ✅ Disponível | Redshift provisionado usa a VPC default |

**Conclusão**: todo o desenho cabe nas permissões do Learner Lab sem gambiarras.

---

## Por que isso é legal pedagogicamente

1. **A mesma pergunta, respostas diferentes** — o aluno vê concretamente que "o número" depende da modelagem, não só dos dados. Isso muda a forma dele conversar com stakeholders pelo resto da carreira.
2. **Narrativa de evolução de negócio** — o Lab 03.2 força o aluno a sentir a dor de modelar para o hoje e precisar acomodar o amanhã. Essa é a realidade de 100% dos warehouses em produção.
3. **Resultados determinísticos dentro de cada modelagem** — o aluno A e o aluno B, rodando a **mesma modelagem** com o **mesmo dataset**, obtêm **o mesmo número**. Diferenças entre modelagens são pedagógicas; diferenças entre colegas são bugs.
4. **Infra como código desde o dia 1** — alunos saem sabendo que cluster Redshift não é algo mágico que alguém clica para subir; é código versionado.
5. **Foco no que importa** — o aluno gasta tempo pensando em grain, SCD, trade-offs de distkey e negociação com o negócio, não clicando em wizard da AWS.
6. **Conecta com os labs anteriores** — S3 (Aula 1), Iceberg (Aula 2), Redshift (Aula 3). O aluno constrói um mental model acumulativo.
7. **Cobre o material conceitual por uso, não por catálogo** — grain, fato/dimensão, SCD, star schema, MPP, distkey, sortkey, COPY, materialized view, lake vs. warehouse. Cada conceito aparece porque resolve uma dor, não porque está na lista.

> [!NOTE]
> **Conceitos do material conceitual não implementados praticamente** (por restrição de ambiente ou escopo de tempo): Spectrum (não listado no PDF), Data Sharing (exige 2 clusters, máx do Learner Lab é 2 nós de um cluster), Zero-ETL (exige integração com RDS Aurora em setup específico), Streaming ingestion (exige Kinesis, não listado no PDF), Redshift ML (custo SageMaker). Todos entram em **discussão teórica guiada** nos READMEs dos labs.

---

## Pré-requisitos para começar este lab

- [x] Laboratório [00-create-codespaces](../00-create-codespaces/README.md) concluído (Codespaces + credenciais AWS)
- [x] Bucket base `base-config-<RM>` criado (usado só para armazenar state do Terraform)
- [x] Sessão ativa no AWS Academy (4h)

> [!TIP]
> Recomendado ter concluído também os labs 01 (S3) e 02 (Iceberg/Athena) antes, pois o aluno já terá contato com `COPY FROM S3`, Glue Catalog e queries analíticas.

---

## Fluxo do aluno, ponta a ponta

```
Início da aula
    │
    ▼
[Codespaces já rodando, credenciais atualizadas]
    │
    ▼
cd 03-Data-Modeling-e-Data-Warehouse/01-provisionamento
terraform init && terraform apply            ← 5-8 min
    │
    ▼
bash scripts/load_tpch.sh                    ← 3-5 min (download + upload + customer_history)
    │
    ▼
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Lab 03.1 — Três modelagens, três respostas (75-90 min)
  cd 02-modelagem-e-carga/
    Etapa A: oltp_mirror       → query-âncora → anota número
    Etapa B: dw_star (SCD1)    → query-âncora → anota número
    Etapa C: dw_star_scd2      → query-âncora → anota número
  Discussão: "qual está certo?"
  Aluno preenche DECISION.md
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    │
    ▼
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Lab 03.2 — Evolução do negócio (60-75 min)
  cd 03-analise-dimensional/
    Evolução 1: nova fórmula de receita (MV vs. view)
    Evolução 2: "cliente ativo" (SCD2 vs. snapshot periódico)
    Evolução 3: SLA de dashboard (redistkey + MV agregada)
  Reflexão final: recalcular histórico ou congelar?
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    │
    ▼
Fim da aula
cd ../01-provisionamento && terraform destroy  ← preserva budget
```

---

## Próximos passos (desenvolvimento deste lab)

- [ ] **Validar proposta com o professor** (este README)
- [ ] Criar `01-provisionamento/` com Terraform (main.tf, variables.tf, outputs.tf, versions.tf)
- [ ] Escrever `scripts/load_tpch.sh` incluindo a geração sintética da `customer_history` (reclassificações posteriores a 1995 para ~5% dos clientes)
- [ ] Criar `02-modelagem-e-carga/` com README narrativo + 3 subpastas de SQLs comentados (A, B, C)
- [ ] Criar `03-analise-dimensional/` com README narrativo + 3 subpastas de SQLs comentados (3 evoluções)
- [ ] Criar `DECISION_TEMPLATE.md` para o aluno registrar a escolha no fim do 03.1
- [ ] Adicionar blocos `💡 Clique para entender` em cada SQL, seguindo o padrão dos labs 01 e 02
- [ ] Calcular e documentar os **três números esperados** da query-âncora (gabarito do professor)
- [ ] Testar ponta a ponta em um AWS Academy Learner Lab limpo
- [ ] Atualizar README raiz do repositório com o link para este lab
- [ ] Gravar prints das telas-chave (se o professor preferir — padrão atual é sem prints no SQL)

---

## Referências

- Material conceitual da Aula 3 (Kimball & Ross, Reis & Housley, Kleppmann & Riccomini + documentação AWS)
- [AWS Academy Learner Lab Restrictions](../academy-learner-lab-aws-restrictions.pdf) (anexo consultado para este desenho)
- [Amazon Redshift Serverless docs](https://docs.aws.amazon.com/redshift/latest/mgmt/serverless-whatis.html)
- [Redshift Spectrum with Glue Data Catalog](https://docs.aws.amazon.com/redshift/latest/dg/c-using-spectrum.html)
- [TPC-H Benchmark Specification](https://www.tpc.org/tpch/)
- Lab [02-Open-Table-Format](../02-Open-Table-Format/) (referência de padrão didático)
