# 02.1 - Funcionalidades básicas com Apache Iceberg no Athena

**Antes de começar, execute os passos abaixo para configurar o ambiente caso não tenha feito isso ainda na aula de HOJE: [Preparando Credenciais](../../00-create-codespaces/Inicio-de-aula.md)**

Neste laboratório, você explorará as funcionalidades básicas do Apache Iceberg e aprenderá a criar e modificar tabelas Iceberg com o Amazon Athena.

Observe que o Amazon Athena fornece suporte integrado para o Apache Iceberg, permitindo ler e gravar em tabelas Iceberg sem adicionar dependências ou configurações extras. Isso é válido para tabelas na [especificação Iceberg v2](https://iceberg.apache.org/spec/#version-2-row-level-deletes).

## Principais pontos de aprendizagem

- criar tabelas Iceberg
- inserir dados em uma tabela Iceberg
- atualizar um único registro
- excluir registros de uma tabela Iceberg
- consultar snapshots e histórico
- evoluir o esquema da tabela

## O que você terá ao final

Ao final deste laboratório, você terá criado uma tabela Iceberg no Athena, carregado dados nela, executado operações de `INSERT`, `UPDATE`, `DELETE`, `FOR VERSION AS OF`, `FOR TIMESTAMP AS OF` e mudanças de esquema.

> [!TIP]
> Sempre que encontrar um bloco com o título **💡 Clique para entender**, abra esse trecho. Ele traz explicação detalhada do comando, contexto prático da aula e links oficiais para aprofundamento.

---

## Parte 1 - Pré-requisitos e criação do ambiente

### Resultado esperado desta parte

Ao final desta etapa, o ambiente base do Athena estará pronto para o laboratório.

1. No Codespaces da disciplina, abra um terminal integrado.

![](img/terminal-inicial.png)

2. No terminal, execute o script abaixo para preparar automaticamente o ambiente do laboratório no Athena, baixando os dados TPC-DS, enviando-os ao S3 e criando as tabelas necessárias:

```bash
cd /workspaces/FIAP-Data-Warehouse-Lakehouse-e-Data-Mesh && bash setup_athena_tpcds.sh
```

<details>
<summary><b>💡 Clique para entender: preparo automático do ambiente</b></summary>
<blockquote>

Esse comando executa um script shell que funciona como um orquestrador do laboratório. A ideia é eliminar tarefas repetitivas de preparação para que a aula fique concentrada no comportamento do Apache Iceberg dentro do Athena.

Em um cenário como este, o script normalmente encadeia etapas como:

- preparar variáveis de ambiente e caminhos de trabalho
- disponibilizar os dados TPC-DS usados nos exemplos
- copiar ou organizar esses dados no Amazon S3
- criar o contexto inicial necessário no Athena
- deixar tabelas de apoio, como `tpcds.prepared_customer`, prontas para consulta

Em outras palavras, ele transforma um processo operacional com vários passos em uma única execução controlada.

### Por que isso é importante nesta aula

Se você tivesse que fazer tudo manualmente, gastaria tempo com download, upload, organização de pastas, criação de estruturas e validações que não são o objetivo principal do exercício. Aqui, o foco é entender recursos de tabela aberta, snapshots, alterações transacionais e consumo analítico.

### Como validar que o script cumpriu o papel dele

Depois da execução, os sinais mais comuns de sucesso são:

- o Athena passa a ter acesso ao ambiente que será usado no laboratório
- as tabelas do conjunto TPC-DS preparado ficam disponíveis
- consultas como seleção, inserção e criação de tabelas Iceberg conseguem avançar sem erro de base inexistente

### Padrão de uso desse tipo de automação

Esse é um padrão muito comum em ambientes de engenharia de dados:

1. preparar a infraestrutura mínima
2. carregar ou registrar dados-base
3. executar a camada analítica em cima desse ambiente já organizado

Ou seja, o script não é o objetivo final da prática. Ele é a fundação que permite exercitar o que realmente importa na aula.

Documentação oficial:
- [Usando Apache Iceberg com o Athena](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg.html)
- [Criação de tabelas no Athena](https://docs.aws.amazon.com/athena/latest/ug/create-table.html)
- [TPC-DS como benchmark analítico](https://www.tpc.org/tpcds/)

</blockquote>
</details>

![img/criacao-tabela.png](img/criacao-tabela.png)

> [!IMPORTANT]
> Só siga para a próxima parte depois que esse script terminar com sucesso.

---

## Parte 2 - Configurando o Athena

### Resultado esperado desta parte

Ao final desta etapa, o editor de consultas do Athena estará configurado para salvar resultados no bucket correto.

3. Acesse o [console do Amazon Athena](https://us-east-1.console.aws.amazon.com/athena/home?region=us-east-1#/landing-page).

4. Selecione **Consulte seus dados no console do Athena** e depois **Iniciar editor de consultas**.

![athena_searchbar](img/athena_launch_query_editor.png)

5. Quando estiver dentro do Athena, clique em **Editar configurações** e depois em **Gerenciar**.

![athena_setup](img/athena_initial_setup.png)

![athena_setup1](img/athena_initial_setup1.png)

6. Clique em `Browse S3`, selecione o bucket que inicia com `otfs-aula`, escolha a pasta `athena_res/` e depois clique em `Choose` e `Salvar`.

![athena_reslocation_setup](img/athena_reslocation_setup.png)

![athena_reslocation_setup](img/athena_reslocation_setup1.png)

![athena_reslocation_setup](img/athena_reslocation_setup2.png)

![athena_reslocation_setup](img/athena_reslocation_setup3.png)

7. Volte para a tela do **Editor**.

![athena_editor](img/athena-editor.png)

### Checkpoint

Se você chegou até aqui, então:

- o Athena está acessível
- o editor de consultas está aberto
- o local de saída das consultas foi configurado

---

## Parte 3 - Criando a base Iceberg

### Resultado esperado desta parte

Ao final desta etapa, o banco `athena_iceberg_db` e a tabela `customer_iceberg` estarão criados.

8. Crie o banco de dados:

```sql
create database athena_iceberg_db;
```

![create-iceberg-db](img/create-iceberg-db.png)

9. Crie a tabela Iceberg abaixo. Antes de executar, substitua `<your-account-id>` pelo ID da sua conta atual.

![](img/getIdAccount.png)

```sql
CREATE TABLE athena_iceberg_db.customer_iceberg (
    c_customer_sk INT COMMENT 'unique id',
    c_customer_id STRING,
    c_first_name STRING,
    c_last_name STRING,
    c_email_address STRING)
LOCATION 's3://otfs-aula-<your-account-id>/datasets/athena_iceberg/customer_iceberg'
TBLPROPERTIES (
  'table_type'='iceberg',
  'format'='PARQUET',
  'write_compression'='zstd'
);
```

![Create-iceberg-table](img/create-iceberg-table.png)

![](img/create-iceberg-table-2.png)

<details>
<summary><b>💡 Clique para entender: criação da tabela Iceberg</b></summary>
<blockquote>

Esse é um dos comandos centrais do laboratório, porque ele define tanto a estrutura lógica da tabela quanto o comportamento transacional esperado no data lake.

### Anatomia do comando

A instrução pode ser lida em blocos:

- definição do nome completo da tabela: `athena_iceberg_db.customer_iceberg`
- definição das colunas e tipos de dados
- indicação do caminho físico no S3 com `LOCATION`
- ativação do formato aberto com `TBLPROPERTIES`

### O papel de cada parte

- `LOCATION` aponta para o local onde os arquivos de dados e metadados do Iceberg serão mantidos
- `'table_type'='iceberg'` habilita os recursos de snapshot, evolução de esquema e operações de linha
- `'format'='PARQUET'` escolhe um formato colunar muito eficiente para leitura analítica
- `'write_compression'='zstd'` ajuda a reduzir tamanho de armazenamento e volume lido nas consultas

### O que muda em relação a uma tabela externa tradicional

Em uma tabela simples sobre arquivos no S3, você normalmente pensa apenas em leitura de arquivos. No Iceberg, você passa a ter também:

- controle de versões da tabela
- histórico de mudanças
- suporte a `UPDATE`, `DELETE` e `MERGE`
- evolução de esquema com muito menos impacto operacional

### Padrão mental para interpretar esse comando

Sempre que estiver criando uma tabela Iceberg no Athena, pense em três camadas:

1. esquema lógico que o usuário consulta
2. dados físicos armazenados no S3
3. metadados que conectam uma versão da tabela aos arquivos corretos

É essa camada de metadados que torna possível o time travel e a consistência transacional.

Documentação oficial:
- [Criando tabelas Iceberg no Athena](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg-creating-tables.html)
- [Especificação oficial do Apache Iceberg](https://iceberg.apache.org/spec/)
- [Visão geral do Apache Iceberg](https://iceberg.apache.org/docs/latest/)

</blockquote>
</details>

10. Valide se a tabela foi criada:

```sql
SHOW TABLES IN athena_iceberg_db;
```

![Create-iceberg-table](img/show_tables_in_db.png)

11. Consulte o esquema da tabela:

```sql
DESCRIBE customer_iceberg;
```

![Create-iceberg-table](img/describe_athena_iceberg_table.png)

### O que validar aqui

- o banco `athena_iceberg_db` existe
- a tabela `customer_iceberg` existe
- a tabela ainda está vazia

---

## Parte 4 - Entendendo a estrutura da tabela Iceberg

A estrutura subjacente do Iceberg é organizada em metadados, snapshots, manifestos e arquivos de dados.

![Create-iceberg-table](img/iceberg_underlying_table_structure.png)

Em alto nível:

- cada operação confirmada gera um novo snapshot
- cada alteração relevante gera um novo arquivo de metadados
- a tabela aponta sempre para o metadado mais recente
- os manifestos apontam para os arquivos de dados

As tabelas Athena Iceberg expõem metadados de tabela, como `files`, `manifests`, `history` e `snapshots`.

12. Consulte os arquivos da tabela. Como a tabela ainda não tem dados, o retorno deverá estar vazio.

```sql
SELECT * FROM "athena_iceberg_db"."customer_iceberg$files"
```

![Create-iceberg-table](img/athena_table_files_no_data.png)

13. Consulte os manifestos da tabela:

```sql
SELECT * FROM "athena_iceberg_db"."customer_iceberg$manifests"
```

14. Consulte os snapshots da tabela:

```sql
SELECT * FROM "athena_iceberg_db"."customer_iceberg$snapshots"
```

<details>
<summary><b>💡 Clique para entender: tabelas de metadados do Iceberg</b></summary>
<blockquote>

Essas consultas são extremamente valiosas porque mostram o que está por trás da tabela sem exigir inspeção manual dos arquivos no S3.

### O que cada sufixo revela

- `$files`: lista os arquivos de dados efetivamente considerados pela versão atual da tabela
- `$manifests`: mostra os manifestos que agrupam metadados sobre conjuntos de arquivos
- `$snapshots`: apresenta cada versão materializada da tabela ao longo do tempo
- `$history`: mostra em que momento cada snapshot passou a ser o estado corrente

### Por que isso importa em uma open table format

Em um lakehouse moderno, a tabela não é apenas uma pasta com arquivos. Ela é um conjunto coordenado de dados + metadados + histórico. Quando você consulta essas tabelas auxiliares, está enxergando exatamente esse mecanismo de coordenação.

### Exemplos de uso prático

Esses metadados são úteis para:

- validar se uma carga gerou novos arquivos
- identificar se um `UPDATE` ou `DELETE` criou um novo snapshot
- investigar o histórico de mudanças de uma tabela
- descobrir o `snapshot_id` que será usado em time travel

### Padrão de leitura dos resultados

Se a tabela ainda estiver vazia, é esperado ver pouco ou nenhum retorno. Depois de um `INSERT`, você tende a ver:

- arquivos Parquet em `$files`
- referências Avro em `$manifests`
- um novo registro em `$snapshots`
- a linha correspondente aparecendo em `$history`

Essa é uma das formas mais didáticas de entender que o Iceberg controla estado por versão, e não apenas por presença física de arquivos.

Documentação oficial:
- [Consultar tabelas Iceberg no Athena](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg-table-data.html)
- [Inspeção de metadados no Apache Iceberg](https://iceberg.apache.org/docs/latest/spark-queries/#inspecting-tables)

</blockquote>
</details>

> [!NOTE]
> Neste momento, essas consultas não devem retornar dados relevantes, porque a tabela ainda está vazia.

---

## Parte 5 - Inserindo dados

### Resultado esperado desta parte

Ao final desta etapa, a tabela `customer_iceberg` terá dados carregados a partir de `tpcds.prepared_customer`.

15. Insira os registros na tabela:

```sql
INSERT INTO athena_iceberg_db.customer_iceberg
SELECT * FROM tpcds.prepared_customer
```

A execução deve terminar com a mensagem **Consulta bem-sucedida**.

<details>
<summary><b>💡 Clique para entender: carregamento com INSERT INTO ... SELECT</b></summary>
<blockquote>

Esse padrão de comando é um dos mais importantes em pipelines analíticos. Ele lê dados de uma origem já preparada e os grava em uma tabela de destino mantendo o formato e as propriedades do Iceberg.

### O que está acontecendo aqui

- a origem é `tpcds.prepared_customer`
- o destino é `athena_iceberg_db.customer_iceberg`
- cada linha selecionada é convertida em arquivos de dados controlados pelo Iceberg
- ao final, um novo snapshot é criado para representar o novo estado da tabela

### Por que esse padrão é tão usado

`INSERT INTO ... SELECT` é a base de muitos processos de:

- ingestão batch
- transformação de dados entre camadas
- preenchimento inicial de tabelas analíticas
- migração de dados para formatos de tabela abertos

### O que observar depois da execução

Após rodar o comando, vale conferir:

- se a contagem da tabela bate com o volume esperado
- se o caminho no S3 passou a ter arquivos em `data` e `metadata`
- se surgiu um novo snapshot nas tabelas de metadados

Documentação oficial:
- [INSERT INTO no Athena](https://docs.aws.amazon.com/athena/latest/ug/insert-into.html)
- [Usando tabelas Iceberg no Athena](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg.html)

</blockquote>
</details>

16. Consulte os primeiros registros:

```sql
select * from athena_iceberg_db.customer_iceberg limit 10;
```

![iceberg-test-query](img/query-iceberg-table.png)

17. Conte o total de registros:

```sql
select count(*) from athena_iceberg_db.customer_iceberg;
```

O resultado esperado é **2.000.000** registros.

### Checkpoint

Se você chegou até aqui, então:

- a tabela foi carregada com sucesso
- já existem arquivos de dados e metadados associados a ela

---

## Parte 6 - Explorando dados e metadados no S3

18. No local da tabela no [Amazon S3](https://us-east-1.console.aws.amazon.com/s3/home?region=us-east-1), abra:

`s3://otfs-aula-<your-account-id>/datasets/athena_iceberg/customer_iceberg/`

Você deverá ver duas pastas:

- `data`
- `metadata`

A pasta `data` contém os dados em Parquet, e a pasta `metadata` contém os arquivos de metadados.

Tipos de arquivo esperados em `metadata`:

- arquivos `.metadata.json`
- listas de manifesto `*-m*.avro`
- manifestos `snap-*.avro`

Pasta de metadados:

![iceberg-test-query](img/iceberg_table_metadata_s3_folder.png)

Pasta de dados:

![iceberg-test-query](img/iceberg_table_data_s3_folder.png)

19. Liste os arquivos da tabela:

```sql
SELECT * FROM "athena_iceberg_db"."customer_iceberg$files"
```

20. Liste os manifestos:

```sql
SELECT * FROM "athena_iceberg_db"."customer_iceberg$manifests"
```

21. Consulte o histórico:

```sql
SELECT * FROM "athena_iceberg_db"."customer_iceberg$history"
```

22. Consulte os snapshots:

```sql
SELECT * FROM "athena_iceberg_db"."customer_iceberg$snapshots"
```

### O que observar

- em `files`, caminhos de arquivos `.parquet`
- em `manifests`, caminhos de arquivos `.avro`
- em `history` e `snapshots`, valores como `snapshot_id`, `parent_id` e `manifest_list`

---

## Parte 7 - Atualizando registros

### Resultado esperado desta parte

Ao final desta etapa, o registro do cliente com `c_customer_sk = 15` terá sido corrigido.

23. Consulte o registro do cliente:

```sql
select * from athena_iceberg_db.customer_iceberg
WHERE c_customer_sk = 15
```

Observe que `c_last_name` e `c_email_address` estão `null`.

24. Atualize o registro:

```sql
UPDATE athena_iceberg_db.customer_iceberg
SET c_last_name = 'John', c_email_address = 'johnTonya@abx.com'
WHERE c_customer_sk = 15
```

A consulta deve terminar com **Consulta bem-sucedida**.

<details>
<summary><b>💡 Clique para entender: UPDATE em tabela Iceberg</b></summary>
<blockquote>

Esse comando faz uma correção pontual de dado, algo muito comum em cenários reais de lakehouse quando há enriquecimento, ajuste cadastral ou retificação operacional.

### Estrutura lógica do comando

- `UPDATE ... SET ...` define quais colunas serão alteradas
- `WHERE c_customer_sk = 15` restringe a mudança a um único registro

Sem esse filtro, o impacto seria muito maior. Em ambiente analítico, esse cuidado é fundamental.

### O que o Iceberg faz internamente

O comportamento relevante aqui não é só o SQL em si, mas a forma como o Iceberg registra a alteração. Em vez de depender de uma reescrita completa da tabela, ele controla a modificação pela camada de metadados e pelos arquivos relacionados ao snapshot novo.

Isso traz benefícios como:

- consistência transacional
- rastreabilidade da mudança
- possibilidade de consultar o estado anterior por time travel
- menor impacto operacional comparado a abordagens mais rígidas

### Padrões de uso

Esse tipo de atualização costuma ser usado para:

- corrigir atributos nulos ou incorretos
- preencher dados de cadastro depois de uma etapa de enriquecimento
- aplicar ajustes vindos de sistemas operacionais

### Boa prática

Sempre valide o registro antes e depois do `UPDATE`, exatamente como o laboratório propõe. Esse padrão reduz risco de alteração indevida e ajuda a ensinar comportamento transacional de tabela aberta.

Documentação oficial:
- [UPDATE em tabelas Iceberg no Athena](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg-update.html)
- [Boas práticas de escrita com Apache Iceberg na AWS](https://docs.aws.amazon.com/prescriptive-guidance/latest/apache-iceberg-on-aws/best-practices-write.html)

</blockquote>
</details>

25. Valide a alteração:

```sql
select * from athena_iceberg_db.customer_iceberg
WHERE c_customer_sk = 15
```

Agora o sobrenome e o e-mail devem aparecer preenchidos.

### Observação técnica

Athena usa [merge-on-read](https://docs.aws.amazon.com/pt_br/prescriptive-guidance/latest/apache-iceberg-on-aws/best-practices-write.html) para operações `UPDATE`.

Na prática, isso significa que:

- ele grava arquivos de exclusão por posição
- grava também as linhas atualizadas
- evita reescrever arquivos inteiros desnecessariamente

26. Verifique o impacto da operação na camada de dados:

```sql
SELECT * FROM "athena_iceberg_db"."customer_iceberg$files"
```

![iceberg-test-query](img/data_file_path_after_update.png)

> [!TIP]
> Você pode identificar novos arquivos observando o `LastModified` no S3.

---

## Parte 8 - Excluindo registros

### Resultado esperado desta parte

Ao final desta etapa, o registro do cliente com `c_customer_sk = 15` terá sido removido da visualização atual da tabela.

27. Exclua o registro:

```sql
delete from athena_iceberg_db.customer_iceberg
WHERE c_customer_sk = 15
```

A consulta deve terminar com **Consulta bem-sucedida**.

28. Valide a remoção:

```sql
SELECT * FROM athena_iceberg_db.customer_iceberg WHERE c_customer_sk = 15
```

O resultado esperado é **Nenhum resultado**.

### Observação técnica

Athena também usa `merge-on-read` para `DELETE`, criando arquivos de exclusão baseados em posição em vez de reescrever todos os arquivos de dados.

<details>
<summary><b>💡 Clique para entender: DELETE em tabela Iceberg</b></summary>
<blockquote>

O `DELETE` remove o registro da visão atual da tabela, mas o aprendizado mais importante aqui é entender que, no Iceberg, exclusão não significa simplesmente apagar um arquivo inteiro do S3.

### O que esse comando representa

Você está dizendo ao mecanismo de tabela que determinado registro não deve mais aparecer no estado corrente. A tabela então gera um novo snapshot coerente com essa remoção.

### Por que isso é poderoso

Esse comportamento permite:

- exclusões mais seguras em ambiente analítico
- manutenção de histórico para auditoria
- integração com fluxos de correção e conformidade de dados

### Padrão prático

Em projetos reais, `DELETE` costuma aparecer em situações como:

- remoção de registros inválidos
- atendimento a regras de governança
- exclusão de duplicidades
- correções de cargas mal executadas

### Relação com time travel

Mesmo após o `DELETE`, uma versão anterior da tabela ainda pode ser consultada por snapshot ou timestamp. Isso ajuda muito em investigação, rollback lógico e validação de mudanças.

Documentação oficial:
- [DELETE em tabelas Iceberg no Athena](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg-delete.html)
- [Operações de linha no Apache Iceberg](https://iceberg.apache.org/spec/#row-level-deletes)

</blockquote>
</details>

---

## Parte 9 - Time travel

### Resultado esperado desta parte

Ao final desta etapa, você terá consultado versões anteriores da tabela usando snapshot e timestamp.

29. Consulte o histórico da tabela:

```sql
SELECT * FROM "athena_iceberg_db"."customer_iceberg$history"
order by made_current_at;
```

![iceberg-test-query](img/iceberg_table_history.png)

Você deverá ver 3 momentos principais:

- inserção inicial
- atualização
- exclusão

30. Substitua `5418594889737463157` pelo `snapshot_id` da linha correspondente ao segundo snapshot e consulte a tabela naquele ponto do tempo:

```sql
select * from athena_iceberg_db.customer_iceberg
FOR VERSION AS OF  5418594889737463157
WHERE c_customer_sk = 15
```

O resultado deve mostrar o registro do cliente Tonya.

31. Agora faça a mesma ideia usando timestamp. Substitua o timestamp abaixo pelo valor de `made_current_at` da linha correta no histórico:

```sql
select * from athena_iceberg_db.customer_iceberg
FOR TIMESTAMP AS OF TIMESTAMP '2024-04-16 17:21:49.771 UTC'
WHERE c_customer_sk = 15
```

Novamente, o resultado deve mostrar o registro do cliente Tonya.

<details>
<summary><b>💡 Clique para entender: time travel com snapshot e timestamp</b></summary>
<blockquote>

Time travel é um dos recursos mais característicos do Iceberg. Ele permite consultar a tabela como ela estava em um momento anterior, sem restaurar backup e sem alterar o estado atual.

### Duas formas de navegar no passado

- `FOR VERSION AS OF`: usa um identificador exato de snapshot
- `FOR TIMESTAMP AS OF`: usa um ponto no tempo, e o mecanismo resolve qual snapshot estava ativo naquele instante

### Quando usar cada uma

Use `FOR VERSION AS OF` quando você quer precisão máxima, por exemplo em auditoria técnica. Use `FOR TIMESTAMP AS OF` quando a pergunta é temporal, como “como a tabela estava antes da exclusão?”.

### Exemplos de situações reais

Esse recurso é muito útil para:

- investigar quando uma informação mudou
- comparar o antes e o depois de um `UPDATE` ou `DELETE`
- validar resultados de uma carga incremental
- recuperar contexto histórico para auditoria ou debugging

### Padrão mental importante

O Iceberg não precisa clonar a tabela inteira para fazer isso. Ele usa a cadeia de snapshots e metadados para reconstruir a visão correta daquele ponto no tempo.

É exatamente por isso que consultar `history` e `snapshots` antes dessa etapa ajuda tanto na compreensão do laboratório.

Documentação oficial:
- [Time travel em tabelas Iceberg no Athena](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg-time-travel-and-version-travel-queries.html)
- [Time travel no Apache Iceberg](https://iceberg.apache.org/docs/latest/spark-queries/#time-travel)

</blockquote>
</details>

---

## Parte 10 - Evolução do esquema

### Resultado esperado desta parte

Ao final desta etapa, a tabela terá uma coluna renomeada e uma nova coluna adicionada, sem reescrita dos arquivos de dados.

As mudanças de esquema no Iceberg são alterações de metadados. Em geral, os arquivos de dados não precisam ser recriados.

32. Consulte os arquivos de dados da tabela:

```sql
SELECT * FROM "athena_iceberg_db"."customer_iceberg$files"
```

Anote o caminho e o nome do arquivo.

33. Renomeie a coluna `c_email_address` para `email`:

```sql
ALTER TABLE athena_iceberg_db.customer_iceberg
change column c_email_address email STRING
```

34. Consulte os arquivos novamente:

```sql
SELECT * FROM "athena_iceberg_db"."customer_iceberg$files"
```

Observe que não há novos arquivos de dados criados por causa da mudança de esquema.

35. Valide o novo esquema:

```sql
DESCRIBE customer_iceberg;
```

36. Adicione uma nova coluna chamada `c_birth_date`:

```sql
ALTER TABLE athena_iceberg_db.customer_iceberg ADD COLUMNS (c_birth_date int)
```

37. Valide novamente:

```sql
DESCRIBE customer_iceberg;
```

38. Consulte a tabela com a nova coluna:

```sql
SELECT *
FROM athena_iceberg_db.customer_iceberg
LIMIT 10
```

A nova coluna deverá aparecer com valores `null` para os registros já existentes.

<details>
<summary><b>💡 Clique para entender: evolução de esquema no Iceberg</b></summary>
<blockquote>

Evolução de esquema é a capacidade de adaptar a estrutura da tabela ao longo do tempo sem quebrar todo o pipeline analítico. Em data platforms reais, isso acontece o tempo todo.

### O que este laboratório mostra

Aqui você realiza duas mudanças clássicas:

- renomear uma coluna existente
- adicionar uma nova coluna ao esquema

### Por que isso é relevante

Em modelos tradicionais, mudanças de esquema podem exigir cópia de dados, recriação de tabelas ou ajustes operacionais maiores. No Iceberg, muitas dessas alterações são registradas na camada de metadados, o que reduz custo e complexidade.

### O que esperar depois da alteração

- o novo nome aparece no `DESCRIBE`
- a nova coluna passa a existir na leitura da tabela
- linhas antigas aparecem com `null` na coluna recém-adicionada, porque esses registros foram gravados antes da mudança

### Cenários reais

Esse recurso é valioso quando:

- um sistema de origem passou a fornecer um novo atributo
- um nome de coluna precisa ficar mais claro para consumo analítico
- o modelo precisa evoluir sem interromper consultas existentes

Documentação oficial:
- [Usando Apache Iceberg com o Athena](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg.html)
- [Evolução de esquema no Apache Iceberg](https://iceberg.apache.org/docs/latest/evolution/#schema-evolution)

</blockquote>
</details>

---

## Conclusão

Se você chegou até aqui, então já executou:

- criação de banco e tabela Iceberg
- inserção de dados
- leitura de metadados
- atualização de registros
- exclusão de registros
- time travel por snapshot e timestamp
- evolução de esquema

Este laboratório forma a base para os próximos exercícios com funcionalidades mais avançadas do Iceberg.
