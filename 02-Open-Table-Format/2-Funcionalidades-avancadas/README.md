# 02.2 - Funcionalidades avançadas com Apache Iceberg no Athena

**Antes de começar, execute os passos abaixo para configurar o ambiente caso não tenha feito isso ainda na aula de HOJE: [Preparando Credenciais](../../00-create-codespaces/Inicio-de-aula.md)**

Neste laboratório, você explorará funcionalidades avançadas do Apache Iceberg.

Observe que o Amazon Athena fornece suporte integrado para o Apache Iceberg, permitindo ler e gravar em tabelas Iceberg sem dependências adicionais. Isso é válido para [tabelas Iceberg v2](https://iceberg.apache.org/spec/#version-2-row-level-deletes).

## Principais pontos de aprendizagem

- particionamento oculto
- uso de `MERGE INTO`
- otimização de tabelas com `OPTIMIZE`

## O que você terá ao final

Ao final deste laboratório, você terá criado uma tabela particionada, aplicado alterações condicionais com `MERGE` e executado manutenção com `OPTIMIZE`.

> [!TIP]
> Sempre que encontrar um bloco com o título **💡 Clique para entender**, abra esse trecho. Ele destaca a lógica do comando, o comportamento esperado no Athena e o motivo técnico da etapa.

> [!IMPORTANT]
> Este laboratório pressupõe que o banco `athena_iceberg_db` já exista e que o ambiente TPC-DS já tenha sido preparado no Athena.

## Pré-requisitos rápidos

Antes de seguir, confirme se:

- o laboratório básico anterior já foi concluído
- o banco `athena_iceberg_db` já existe
- a tabela de origem `tpcds.prepared_web_sales` está acessível no Athena
- o local de saída das consultas no Athena já foi configurado

---

## Parte 1 - Particionamento oculto

### Resultado esperado desta parte

Ao final desta etapa, você terá criado uma tabela Iceberg particionada por ano e validado que o Athena aplica pruning automaticamente durante a leitura.

O particionamento oculto do Iceberg é uma melhoria sobre a abordagem tradicional do Hive. Em vez de o consumidor precisar conhecer explicitamente as colunas e valores de partição, o Iceberg consegue derivar e usar essas informações automaticamente.

Neste exercício, a tabela será particionada por ano com base em `ws_sales_time`.

1. Crie a tabela `web_sales_iceberg`. Antes de executar, substitua `<your-account-id>` pelo ID da sua conta atual.

```sql
CREATE TABLE athena_iceberg_db.web_sales_iceberg (
    ws_order_number INT,
    ws_item_sk INT,
    ws_quantity INT,
    ws_sales_price DOUBLE,
    ws_warehouse_sk INT,
    ws_sales_time TIMESTAMP)
PARTITIONED BY (year(ws_sales_time))
LOCATION 's3://otfs-aula-<your-account-id>/datasets/athena_iceberg/web_sales_iceberg'
TBLPROPERTIES (
  'table_type'='iceberg',
  'format'='PARQUET',
  'write_compression'='ZSTD'
);
```

A consulta deve terminar com **Consulta bem-sucedida**.

<details>
<summary><b>💡 Clique para entender: particionamento oculto no Iceberg</b></summary>
<blockquote>

Esse ponto é um dos grandes diferenciais do Iceberg em relação a abordagens mais antigas de data lake.

### O que significa particionar por ano

Ao usar `PARTITIONED BY (year(ws_sales_time))`, você está dizendo que a organização física da tabela deve considerar o ano derivado da coluna de timestamp. Isso ajuda o mecanismo a evitar leitura desnecessária quando a consulta filtra intervalo de datas.

### Por que o termo “oculto” é importante

No modelo tradicional do Hive, o usuário muitas vezes precisava conhecer explicitamente a coluna de partição e até adaptar a consulta pensando no layout físico. No Iceberg, a tabela guarda essa lógica de forma mais inteligente.

Ou seja:

- você consulta pela coluna de negócio original
- o Iceberg resolve internamente a transformação de partição
- o Athena aproveita isso para fazer pruning e ler menos dados

### Benefício prático

Esse modelo traz três ganhos principais:

- melhor desempenho de leitura
- menos acoplamento entre consulta e layout físico
- maior facilidade para evoluir a estratégia de particionamento no futuro

### Padrão de uso

Sempre que uma tabela analítica possui filtros frequentes por tempo, país, categoria ou outra dimensão de alta seletividade, vale avaliar uma estratégia de particionamento coerente com esse padrão de consulta.

Documentação oficial:
- [Particionamento em tabelas Iceberg no Athena](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg-creating-tables.html)
- [Partitioning no Apache Iceberg](https://iceberg.apache.org/docs/latest/partitioning/)

</blockquote>
</details>

2. Insira os registros com ano anterior a 2001:

```sql
INSERT INTO athena_iceberg_db.web_sales_iceberg
SELECT * FROM tpcds.prepared_web_sales where year(ws_sales_time) < 2001;
```

3. Verifique se a tabela possui dados:

```sql
SELECT *
FROM athena_iceberg_db.web_sales_iceberg
LIMIT 10;
```

4. Consulte os arquivos de dados para validar a partição por ano:

```sql
SELECT * FROM "athena_iceberg_db"."web_sales_iceberg$files"
```

![Create-iceberg-table](img/partitioned_by_year.png)

5. Agora execute a consulta abaixo para verificar o comportamento do particionamento oculto:

```sql
SELECT COUNT(DISTINCT(ws_order_number)) AS num_orders
FROM athena_iceberg_db.web_sales_iceberg
WHERE ws_sales_time >= TIMESTAMP '2000-01-01 00:00:00' AND ws_sales_time < TIMESTAMP '2000-02-01 00:00:00'
```

Ao verificar as estatísticas da consulta, as **Linhas de entrada** deverão refletir apenas a partição relevante.

![Hidden-stats](img/query_hidden_partition_stats.png)

6. Confirme a distribuição por ano:

```sql
SELECT YEAR(ws_sales_time) as year, COUNT(*) AS records_per_year
FROM athena_iceberg_db.web_sales_iceberg
GROUP BY YEAR(ws_sales_time)
```

![Hidden-confirmation](img/query_hidden_partition_confirm.png)

7. Se quiser inspecionar o plano de execução e o volume de leitura, execute:

```sql
EXPLAIN ANALYZE SELECT COUNT(DISTINCT(ws_order_number)) AS num_orders
FROM athena_iceberg_db.web_sales_iceberg
WHERE ws_sales_time >= TIMESTAMP '2000-01-01 00:00:00' AND ws_sales_time < TIMESTAMP '2000-02-01 00:00:00'
```

![Hidden-analyze](img/query_hidden_partition_analyze.png)

<details>
<summary><b>💡 Clique para entender: EXPLAIN ANALYZE neste contexto</b></summary>
<blockquote>

Aqui o objetivo é confirmar, com evidência, se o Athena está lendo apenas a parte necessária da tabela particionada.

### O que o comando faz

- `EXPLAIN` mostra o plano de execução que a engine pretende usar
- `EXPLAIN ANALYZE` executa a consulta e devolve estatísticas reais do processamento

Ou seja, ele deixa de ser apenas uma hipótese de plano e passa a mostrar como a consulta se comportou de verdade.

### Por que isso importa neste laboratório

Como a tabela foi particionada por ano, queremos observar se um filtro temporal faz pruning corretamente. Se estiver tudo certo, o Athena evita varrer anos que não participam da análise.

### O que observar na prática

Ao inspecionar o resultado, concentre-se em pontos como:

- quantidade de dados lidos
- linhas de entrada por operador
- presença de `TableScan`, `Filter` e `Aggregate`
- diferença entre uma consulta com filtro de data e outra sem filtro

### Padrão de interpretação

Se a consulta acessa muito mais dados do que o necessário, isso pode indicar:

- filtro mal definido
- estratégia de partição inadequada
- baixa seletividade na condição

Esse tipo de leitura do plano é essencial para análise de performance em lakehouse.

Documentação oficial:
- [EXPLAIN no Athena](https://docs.aws.amazon.com/athena/latest/ug/athena-explain-statement.html)
- [Como entender o resultado do EXPLAIN](https://docs.aws.amazon.com/athena/latest/ug/athena-explain-statement-understanding.html)

</blockquote>
</details>

### Checkpoint

Se você chegou até aqui, então:

- a tabela `web_sales_iceberg` existe
- ela está particionada por ano
- o Athena já consegue evitar leitura desnecessária de partições

---

## Parte 2 - Atualizar, excluir ou inserir linhas condicionalmente com MERGE

### Resultado esperado desta parte

Ao final desta etapa, você terá usado uma tabela auxiliar para aplicar inserções, atualizações e exclusões condicionais na tabela de destino.

O comando [`MERGE INTO`](https://docs.aws.amazon.com/athena/latest/ug/merge-into-statement.html) é transacional e combina `UPDATE`, `DELETE` e `INSERT` em uma única instrução.

### Etapa 2.1 - Criando a tabela auxiliar

8. Crie a tabela `merge_table`. Antes de executar, substitua `<your-account-id>` pelo ID da sua conta atual.

```sql
CREATE TABLE athena_iceberg_db.merge_table (
    ws_order_number INT,
    ws_item_sk INT,
    ws_quantity INT,
    ws_sales_price DOUBLE,
    ws_warehouse_sk INT,
    ws_sales_time TIMESTAMP,
    operation string)
PARTITIONED BY (year(ws_sales_time))
LOCATION 's3://otfs-aula-<your-account-id>/datasets/athena_iceberg/merge_table'
TBLPROPERTIES (
  'table_type'='iceberg',
  'format'='PARQUET',
  'write_compression'='ZSTD'
);
```

### Etapa 2.2 - Preparando os dados de merge

A coluna `operation` identifica o que cada linha representa:

- `U`: update
- `I`: insert
- `D`: delete

9. Insira as linhas que representarão atualização:

```sql
INSERT INTO athena_iceberg_db.merge_table
SELECT ws_order_number, ws_item_sk, ws_quantity, ws_sales_price, 16 AS ws_warehouse_sk, ws_sales_time, 'U' as operation
FROM tpcds.prepared_web_sales where year(ws_sales_time) = 2000 AND ws_warehouse_sk = 10
```

10. Insira as linhas que representarão inserção:

```sql
INSERT INTO athena_iceberg_db.merge_table
SELECT ws_order_number, ws_item_sk, ws_quantity, ws_sales_price, ws_warehouse_sk, ws_sales_time, 'I' as operation
FROM tpcds.prepared_web_sales where year(ws_sales_time) = 2001
```

11. Insira as linhas que representarão exclusão:

```sql
INSERT INTO athena_iceberg_db.merge_table
SELECT ws_order_number, ws_item_sk, ws_quantity, ws_sales_price, ws_warehouse_sk, ws_sales_time, 'D' as operation
FROM tpcds.prepared_web_sales where year(ws_sales_time) = 1999 AND ws_warehouse_sk = 9
```

12. Valide se a tabela auxiliar contém registros para os 3 tipos de operação:

```sql
select operation, count(*) as num_records
from athena_iceberg_db.merge_table
group by operation
```

### Etapa 2.3 - Aplicando o merge

13. Aplique as alterações da `merge_table` na `web_sales_iceberg`:

```sql
MERGE INTO athena_iceberg_db.web_sales_iceberg t
USING athena_iceberg_db.merge_table s
    ON t.ws_order_number = s.ws_order_number AND t.ws_item_sk = s.ws_item_sk
WHEN MATCHED AND s.operation like 'D' THEN DELETE
WHEN MATCHED AND s.operation like 'U' THEN UPDATE SET ws_order_number = s.ws_order_number, ws_item_sk = s.ws_item_sk, ws_quantity = s.ws_quantity, ws_sales_price = s.ws_sales_price, ws_warehouse_sk = s.ws_warehouse_sk, ws_sales_time = s.ws_sales_time
WHEN NOT MATCHED THEN INSERT (ws_order_number, ws_item_sk, ws_quantity, ws_sales_price, ws_warehouse_sk, ws_sales_time) VALUES (s.ws_order_number, s.ws_item_sk, s.ws_quantity, s.ws_sales_price, s.ws_warehouse_sk, s.ws_sales_time)
```

<details>
<summary><b>💡 Clique para entender: comando MERGE INTO</b></summary>
<blockquote>

O `MERGE INTO` é um comando muito importante em arquitetura analítica moderna porque concentra atualização, inserção e exclusão em uma única instrução transacional.

### Como ler a estrutura do comando

- a tabela de destino recebe um alias, aqui `t`
- a tabela de origem recebe um alias, aqui `s`
- a cláusula `ON` define a chave de correspondência entre origem e destino
- as cláusulas `WHEN MATCHED` e `WHEN NOT MATCHED` definem a ação para cada caso

### O padrão usado neste laboratório

A coluna `operation` funciona como um indicador semântico:

- `U` representa update
- `I` representa insert
- `D` representa delete

Com isso, uma tabela auxiliar controla todo o comportamento do merge.

### Por que esse padrão é tão usado

Esse desenho aparece com frequência em:

- ingestão incremental de dados
- processos de CDC
- sincronização entre camada operacional e camada analítica
- atualização periódica de dimensões e fatos

### Boa prática conceitual

O mais importante para um `MERGE` seguro é garantir que a condição do `ON` identifique corretamente os registros. Em outras palavras, a chave de correspondência precisa refletir a identidade do dado que está sendo sincronizado.

Documentação oficial:
- [MERGE INTO no Athena](https://docs.aws.amazon.com/athena/latest/ug/merge-into-statement.html)
- [Operações de linha no Apache Iceberg](https://iceberg.apache.org/spec/#row-level-deletes)

</blockquote>
</details>

### Etapa 2.4 - Validando o merge

14. Confirme que existem dados para o ano de 2001:

```sql
SELECT YEAR(ws_sales_time) AS year, COUNT(*) as records_per_year
FROM athena_iceberg_db.web_sales_iceberg
GROUP BY (YEAR(ws_sales_time))
ORDER BY year
```

15. Confirme que, no ano 2000, os registros do depósito 10 foram atualizados para o depósito 16:

```sql
SELECT ws_warehouse_sk, COUNT(*) as records_per_warehouse
FROM athena_iceberg_db.web_sales_iceberg
WHERE YEAR(ws_sales_time) = 2000
GROUP BY ws_warehouse_sk
ORDER BY ws_warehouse_sk
```

16. Confirme que, no ano 1999, não restaram entradas com depósito 9:

```sql
SELECT ws_warehouse_sk, COUNT(*) as records_per_warehouse
FROM athena_iceberg_db.web_sales_iceberg
WHERE YEAR(ws_sales_time) = 1999
GROUP BY ws_warehouse_sk
ORDER BY ws_warehouse_sk
```

> [!TIP]
> Você também pode consultar a tabela de snapshots depois do `MERGE` e observar um novo snapshot com `operation = overwrite`.

### Checkpoint

Se você chegou até aqui, então:

- a tabela auxiliar foi criada
- os registros de update, insert e delete foram preparados
- o `MERGE` foi aplicado com sucesso
- a tabela de destino foi alterada conforme esperado

---

## Parte 3 - Otimizando tabelas Iceberg

### Resultado esperado desta parte

Ao final desta etapa, você terá usado `OPTIMIZE` para reorganizar os arquivos da tabela e melhorar a eficiência de leitura.

À medida que os dados se acumulam em uma tabela Iceberg, as consultas podem se tornar menos eficientes por causa da quantidade de arquivos a serem abertos e do custo extra de aplicar arquivos de exclusão.

O comando [`OPTIMIZE`](https://docs.aws.amazon.com/athena/latest/ug/optimize-statement.html) ajuda a:

- compactar arquivos pequenos em arquivos maiores
- mesclar arquivos de exclusão com arquivos de dados

17. Observe o estado atual dos arquivos da tabela:

```sql
SELECT * FROM "athena_iceberg_db"."web_sales_iceberg$files";
```

![Create-iceberg-table](img/file_list_before_compaction.png)

18. Execute a otimização na tabela inteira:

```sql
OPTIMIZE athena_iceberg_db.web_sales_iceberg REWRITE DATA USING BIN_PACK;
```

A consulta deve terminar com **Consulta bem-sucedida**.

<details>
<summary><b>💡 Clique para entender: comando OPTIMIZE</b></summary>
<blockquote>

`OPTIMIZE` é um comando de manutenção física da tabela. Ele não muda a lógica do dado de negócio, mas melhora a forma como os arquivos ficam organizados para leitura posterior.

### O problema que ele resolve

Ao longo do tempo, uma tabela Iceberg pode acumular:

- muitos arquivos pequenos
- arquivos resultantes de updates e deletes
- fragmentação que aumenta o custo de leitura

Quando isso acontece, a consulta continua correta, mas pode ficar menos eficiente.

### O que significa `REWRITE DATA USING BIN_PACK`

A estratégia de bin pack tenta reorganizar os arquivos em grupos mais equilibrados, reduzindo a fragmentação e melhorando o aproveitamento das leituras futuras.

### Quando esse comando costuma fazer sentido

Esse padrão aparece bastante depois de:

- várias cargas incrementais pequenas
- operações frequentes de `MERGE`, `UPDATE` ou `DELETE`
- períodos em que a tabela começou a sofrer degradação de performance

### O que comparar antes e depois

O laboratório acerta ao pedir a inspeção dos arquivos antes e depois do `OPTIMIZE`. É exatamente assim que se valida o efeito operacional da manutenção:

- quantidade de arquivos
- distribuição por partição
- consolidação do estado da tabela

Documentação oficial:
- [OPTIMIZE no Athena](https://docs.aws.amazon.com/athena/latest/ug/optimize-statement.html)
- [Manutenção de tabelas no Apache Iceberg](https://iceberg.apache.org/docs/latest/maintenance/)

</blockquote>
</details>

19. Consulte novamente os arquivos da tabela:

```sql
SELECT * FROM "athena_iceberg_db"."web_sales_iceberg$files";
```

![Create-iceberg-table](img/file_list_after_compression.png)

### O que observar

Compare a situação antes e depois:

- quantidade total de arquivos
- quantidade de registros por partição
- consolidação de arquivos de dados e arquivos de exclusão

Em especial:

- `ws_sales_time_year=1998`: sem mudanças relevantes
- `ws_sales_time_year=1999`: redução por causa das exclusões
- `ws_sales_time_year=2000`: consolidação após updates
- `ws_sales_time_year=2001`: manutenção do estado esperado para as inserções

20. Consulte os snapshots da tabela:

```sql
SELECT * FROM "athena_iceberg_db"."web_sales_iceberg$snapshots";
```

Você deverá ver um novo snapshot com `operation = replace`.

21. Se quiser otimizar apenas uma partição específica, use:

```sql
OPTIMIZE athena_iceberg_db.web_sales_iceberg REWRITE DATA USING BIN_PACK
where year(ws_sales_time) = 2000
```

### Ajustando propriedades de otimização

Você também pode controlar o tamanho dos arquivos e os limites usados pelo processo de compactação por meio de propriedades de tabela.

Exemplo durante a criação:

```sql
CREATE TABLE athena_iceberg_db.web_sales_iceberg (
  ws_order_number INT,
  ws_item_sk INT,
  ws_quantity INT,
  ws_sales_price DOUBLE,
  ws_warehouse_sk INT,
  ws_sales_time TIMESTAMP)
  PARTITIONED BY (year(ws_sales_time))
  LOCATION 's3://otfs-aula-<your-account-id>/datasets/athena_iceberg/web_sales_iceberg'
  TBLPROPERTIES (
  'table_type'='iceberg',
  'format'='PARQUET',
  'write_compression'='ZSTD',
  'write_target_data_file_size_bytes'='346870912',
  'optimize_rewrite_delete_file_threshold'='16',
  'optimize_rewrite_data_file_threshold'='16'
);
```

Exemplo depois da criação:

```sql
ALTER TABLE athena_iceberg_db.web_sales_iceberg SET TBLPROPERTIES (
'write_target_data_file_size_bytes'='346870912',
'optimize_rewrite_delete_file_threshold'='16',
'optimize_rewrite_data_file_threshold'='16'
)
```

---

## Conclusão

Se você chegou até aqui, então já executou:

- particionamento oculto
- leitura eficiente com pruning
- `MERGE INTO` com insert, update e delete
- manutenção com `OPTIMIZE`

Este laboratório mostra como o Iceberg combina governança transacional, desempenho de leitura e manutenção operacional dentro do Athena.
