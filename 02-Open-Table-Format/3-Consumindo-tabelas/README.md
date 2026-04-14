# 02.3 - Consumindo tabelas Iceberg no Athena

**Antes de começar, execute os passos abaixo para configurar o ambiente caso não tenha feito isso ainda na aula de HOJE: [Preparando Credenciais](../../00-create-codespaces/Inicio-de-aula.md)**

Neste laboratório, você explorará como usar o Amazon Athena para consultar tabelas Iceberg.

Observe que o Amazon Athena fornece suporte integrado para o Apache Iceberg, permitindo ler e gravar em tabelas Iceberg sem configurações adicionais. Isso é válido para tabelas na [especificação Iceberg v2](https://iceberg.apache.org/spec/#version-2-row-level-deletes).

## Principais pontos de aprendizagem

- consultar tabelas Iceberg no Athena
- usar `EXPLAIN` e `EXPLAIN ANALYZE`
- criar e consultar views sobre tabelas Iceberg

## O que você precisa já ter pronto

Você consultará as tabelas `web_sales_iceberg` e `customer_iceberg` que foram criadas em laboratórios anteriores de Glue, EMR ou Athena.

> [!TIP]
> Sempre que encontrar um bloco com o título **💡 Clique para entender**, abra esse trecho. Ele foi pensado para ajudar o aluno a interpretar o comando e conectar a prática ao conceito.

> [!IMPORTANT]
> Esta parte depende diretamente dos laboratórios anteriores. Se as tabelas ainda não existirem, volte e conclua os exercícios anteriores antes de continuar.

> [!NOTE]
> Se aparecer erro de tabela inexistente, valide primeiro se o banco selecionado no painel esquerdo do Athena corresponde ao laboratório em que você criou as tabelas.

---

## Parte 1 - Escolhendo o banco correto

### Resultado esperado desta parte

Ao final desta etapa, você terá selecionado o banco correspondente ao laboratório anterior que usou para criar as tabelas.

Esta seção pode ser usada para consultar qualquer uma das tabelas criadas anteriormente.

Selecione no painel esquerdo do Athena o banco correspondente ao ambiente em que você criou as tabelas:

- `glue_iceberg_db`, se criou as tabelas no laboratório de Glue
- `emr_iceberg_db`, se criou as tabelas no laboratório de EMR
- `athena_iceberg_db`, se criou as tabelas no laboratório de Athena

![db_selection](img/select_db_athena.png)

> [!TIP]
> Esse é o ponto de erro mais comum deste laboratório. Se a consulta disser que a tabela não existe, a primeira validação é conferir se o banco selecionado está correto.

---

## Parte 2 - Consultando tabelas Iceberg

### Resultado esperado desta parte

Ao final desta etapa, você terá executado consultas básicas sobre as tabelas Iceberg.

1. Execute a consulta abaixo para consultar um conjunto de dados Iceberg:

```sql
SELECT ws_warehouse_sk, count(distinct(ws_order_number)) as num_orders
FROM web_sales_iceberg
WHERE ws_warehouse_sk in (5,6,10,11)
GROUP BY ws_warehouse_sk
```

2. Verifique a quantidade de registros presentes na tabela de clientes:

```sql
SELECT count(*)
FROM customer_iceberg
```

### Observação importante

As consultas seguem a [especificação de formato Iceberg v2](https://iceberg.apache.org/spec/#format-versioning). Caso a consulta seja executada sobre uma tabela que tenha usado `merge-on-read` — por exemplo, tabelas em `athena_iceberg_db` — os arquivos de exclusão por posição serão mesclados com os arquivos de dados no momento da leitura.

<details>
<summary><b>💡 Clique para entender: consumo de tabelas Iceberg no Athena</b></summary>
<blockquote>

O grande valor aqui é perceber que consumir uma tabela Iceberg é muito mais do que ler um conjunto de arquivos Parquet soltos no S3.

### O que acontece quando você faz um SELECT

Do ponto de vista do aluno, a consulta é apenas SQL. Mas internamente o Athena precisa:

- localizar o snapshot atual da tabela
- identificar os manifestos relevantes
- descobrir quais arquivos de dados pertencem àquela versão
- considerar arquivos de deleção quando existirem operações de linha
- montar a visão consistente que será entregue como resultado

### O que isso muda na prática

Esse mecanismo permite que você tenha em um data lake capacidades típicas de banco analítico moderno, como:

- leitura consistente mesmo após `UPDATE` e `DELETE`
- histórico de versões
- evolução de esquema
- maior governança sobre a tabela

### Padrão mental importante

Em uma open table format, a tabela deixa de ser apenas armazenamento e passa a ser uma estrutura governada por metadados. É isso que viabiliza o conceito de lakehouse.

Documentação oficial:
- [Consultar Apache Iceberg no Athena](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg.html)
- [Especificação oficial do Apache Iceberg](https://iceberg.apache.org/spec/)

</blockquote>
</details>

### Checkpoint

Se você chegou até aqui, então:

- o banco correto foi selecionado
- as tabelas estão acessíveis
- as consultas básicas estão funcionando

---

## Parte 3 - Usando EXPLAIN e EXPLAIN ANALYZE

### Resultado esperado desta parte

Ao final desta etapa, você terá inspecionado o plano de execução e o custo computacional de consultas no Athena.

3. Use `EXPLAIN` para visualizar o plano lógico ou distribuído da consulta:

```sql
EXPLAIN SELECT count(*) FROM customer_iceberg LIMIT 10;
```

4. Use `EXPLAIN ANALYZE` para visualizar o plano de execução distribuído com custo computacional:

```sql
EXPLAIN ANALYZE
SELECT ws_warehouse_sk, count(distinct(ws_order_number)) as num_orders
FROM web_sales_iceberg
WHERE ws_warehouse_sk in (5,6,10,11)
GROUP BY ws_warehouse_sk
```

<details>
<summary><b>💡 Clique para entender: EXPLAIN e EXPLAIN ANALYZE</b></summary>
<blockquote>

Esses comandos são fundamentais para sair do nível “a consulta funciona” e chegar ao nível “eu entendo como a engine está trabalhando”.

### O que é o EXPLAIN

`EXPLAIN` pede ao Athena o plano de execução da consulta. Em vez de retornar o dado de negócio, ele devolve a estratégia planejada pela engine para ler, filtrar, agregar e distribuir o processamento.

Isso ajuda a responder perguntas como:

- a consulta vai varrer a tabela inteira ou só parte dela?
- há filtros sendo aplicados cedo no plano?
- a agregação está acontecendo de forma esperada?

### O que é o EXPLAIN ANALYZE

`EXPLAIN ANALYZE` vai além: ele executa a consulta e mostra estatísticas reais do que aconteceu. É por isso que ele é tão útil para análise de performance.

### Padrões de uso muito comuns

Você pode usar esses comandos para comparar:

- uma consulta com filtro versus sem filtro
- uma versão antes e depois de particionamento
- uma leitura direta da tabela versus uma view analítica

### Exemplo mental para interpretar

Se você filtra por um subconjunto pequeno de dados e o plano ainda mostra leitura muito ampla, isso é um sinal de que o desenho da tabela ou da consulta pode ser melhorado.

Por outro lado, quando o volume lido cai bastante após um filtro seletivo, isso indica que o mecanismo está aproveitando bem a estrutura do Iceberg.

### O que observar nos resultados

Em geral, vale prestar atenção em:

- operadores de leitura, filtro e agregação
- linhas de entrada e saída
- volume processado
- custo total percebido na execução

Documentação oficial:
- [EXPLAIN e EXPLAIN ANALYZE no Athena](https://docs.aws.amazon.com/athena/latest/ug/athena-explain-statement.html)
- [Como entender os resultados do EXPLAIN](https://docs.aws.amazon.com/athena/latest/ug/athena-explain-statement-understanding.html)

</blockquote>
</details>

> [!TIP]
> Use `EXPLAIN` quando quiser validar ou entender a estratégia de execução. Use `EXPLAIN ANALYZE` quando quiser inspecionar custo real de processamento.

---

## Parte 4 - Criando e consultando visualizações

### Resultado esperado desta parte

Ao final desta etapa, você terá criado uma view sobre a tabela Iceberg e consultado seu resultado.

5. Crie a view abaixo:

```sql
CREATE VIEW total_orders_by_warehouse
AS
SELECT ws_warehouse_sk, count(distinct(ws_order_number)) as num_orders
FROM web_sales_iceberg
WHERE ws_warehouse_sk in (5,6,10,11)
GROUP BY ws_warehouse_sk
```

A execução deve terminar com **Consulta bem-sucedida**.

<details>
<summary><b>💡 Clique para entender: comando CREATE VIEW</b></summary>
<blockquote>

Uma view cria uma camada lógica de consumo sobre a tabela original. Ela não copia os dados nem cria um novo conjunto físico de arquivos no S3.

### O que isso resolve

Em vez de cada pessoa escrever sempre a mesma consulta com filtros e agregações, a view encapsula essa lógica em um objeto reutilizável.

### Vantagens práticas

Ela é útil para:

- simplificar análises repetitivas
- esconder complexidade de consultas maiores
- padronizar indicadores para times diferentes
- criar uma camada mais próxima da linguagem de negócio

### Padrão comum em analytics

É muito comum deixar a tabela Iceberg como camada base e construir views por cima para expor métricas, recortes e regras de leitura mais amigáveis.

### O que aprender com este exemplo

Neste caso, a view resume pedidos por depósito. Isso mostra como transformar uma tabela transacionalmente robusta em um artefato mais fácil de consumir no dia a dia analítico.

Documentação oficial:
- [CREATE VIEW no Athena](https://docs.aws.amazon.com/athena/latest/ug/views-console.html)
- [Consultar Apache Iceberg no Athena](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg.html)

</blockquote>
</details>

6. Consulte a view criada:

```sql
SELECT *
FROM total_orders_by_warehouse
```

### Checkpoint final

Se você chegou até aqui, então:

- conseguiu consultar tabelas Iceberg diretamente
- conseguiu analisar consultas com `EXPLAIN` e `EXPLAIN ANALYZE`
- conseguiu criar uma `VIEW` sobre dados Iceberg no Athena

---

## Conclusão

Este laboratório fecha o ciclo de uso das tabelas Iceberg pelo ponto de vista do consumo analítico.

A partir daqui, o aluno já consegue:

- localizar o banco correto
- consultar tabelas Iceberg
- interpretar o plano de execução
- criar abstrações reutilizáveis com `VIEW`
