# 01.1 - Storage de objetos

**Antes de começar, execute os passos abaixo para configurar o ambiente caso não tenha feito isso ainda na aula de HOJE: [Preparando Credenciais](../../00-create-codespaces/Inicio-de-aula.md)**

**Todos os comandos de terminal deste exercício devem ser executados no Codespaces que você criou na configuração inicial.**

## Objetivo do laboratório

Neste laboratório, você vai:

- revisar o bucket criado no setup inicial
- enviar arquivos para o Amazon S3
- testar estratégias de upload para arquivos grandes, médios e pequenos
- comparar abordagens de cópia no S3

## O que você terá ao final

Ao final do exercício, você terá entendido como o comportamento do S3 muda conforme o tamanho do arquivo e conforme a estratégia de transferência usada.

> [!TIP]
> Sempre que encontrar um bloco com o título **💡 Clique para entender**, abra esse trecho. Ele traz uma explicação complementar do comando, do comportamento esperado e do contexto da aula.

---

## Contexto

O Amazon S3 oferece desempenho otimizado para diferentes tamanhos de arquivos, mas requer estratégias específicas para cada caso.

### Arquivos grandes
Para arquivos grandes, o S3 suporta upload multipart e transfer acceleration, que podem reduzir significativamente o tempo de upload. O multipart upload divide arquivos grandes em partes menores, permitindo uploads paralelos e melhorando a eficiência.

### Arquivos pequenos (1 MB)
Para arquivos de aproximadamente 1 MB, o S3 ainda oferece bom desempenho, mas pode-se otimizar ajustando configurações como `max_concurrent_requests` e `multipart_threshold` na AWS CLI.

### Arquivos minúsculos (1 KB ou menos)
O desempenho do S3 para arquivos muito pequenos pode ser desafiador devido ao overhead de cada operação. Nesses casos, técnicas de agrupamento e paralelismo costumam fazer mais diferença.

---

## Parte 1 - Primeiros passos no S3

### Resultado esperado desta parte

Ao final desta etapa, você terá enviado arquivos de exemplo para o bucket base criado no setup.

1. Você irá utilizar o bucket criado no setup da aula anterior. Para verificar se o bucket foi criado, execute o comando abaixo no terminal do Codespaces e confirme se o bucket `base-config-<SEU RM>` existe:

```bash
aws s3 ls
```

![](img/s3-1.png)

2. Popule uma variável chamada `bucket` com o nome do bucket que você criou:

```bash
export bucket=$(aws s3 ls | awk '/base-config-/ {print $3; exit}') && echo $bucket
```

> [!TIP]
> O `echo $bucket` funciona como validação imediata. Se nada for exibido, volte ao setup inicial e confira se o bucket foi criado.

3. Entre na pasta correta para executar o exercício:

```bash
cd /workspaces/FIAP-Data-Warehouse-Lakehouse-e-Data-Mesh/01-Storage/01-Storage-de-Objetos
```

4. Baixe os 3 arquivos CSV que servirão de exemplo para o exercício:

```bash
curl https://perso.telecom-paristech.fr/eagan/class/igr204/data/cereal.csv -o cereal.csv
curl https://perso.telecom-paristech.fr/eagan/class/igr204/data/cars.csv -o car.csv
curl https://perso.telecom-paristech.fr/eagan/class/igr204/data/factbook.csv -o factbook.csv
```

5. Envie os arquivos para o bucket S3 executando os comandos abaixo:

```bash
aws s3 cp car.csv s3://$bucket/car/car.csv

aws s3 cp cereal.csv s3://$bucket/cereal/cereal.csv

aws s3 cp factbook.csv s3://$bucket/factbook/factbook.csv

aws s3 cp factbook.csv s3://$bucket/other/factbook.tst
```

<details>
<summary><b>💡 Clique para entender: comandos de cópia dos arquivos</b></summary>
<blockquote>

O comando `aws s3 cp` é utilizado para copiar arquivos entre o sistema de arquivos local e buckets do Amazon S3, ou entre buckets S3.

Exemplos comuns:

```bash
aws s3 cp arquivo_local.txt s3://meu-bucket/arquivo.txt
aws s3 cp s3://meu-bucket/arquivo.txt arquivo_local.txt
aws s3 cp s3://bucket-origem/arquivo.txt s3://bucket-destino/arquivo.txt
aws s3 cp diretorio/ s3://meu-bucket/diretorio --recursive
```

Ele também permite controlar ACL, filtros, classe de armazenamento, expiração e outras opções.

</blockquote>
</details>

6. Verifique se os arquivos foram enviados corretamente acessando o [painel do S3](https://us-east-1.console.aws.amazon.com/s3/buckets?region=us-east-1&bucketType=general) no console da AWS. Clique no bucket e confirme se os arquivos estão visíveis.

![](img/s3-2.png)

### Checkpoint

Se você chegou até aqui, então:

- o bucket base foi encontrado
- a variável `bucket` foi preenchida
- os arquivos CSV foram enviados para o S3

---

## Parte 2 - Configurando a AWS CLI para testes de performance

### Resultado esperado desta parte

Ao final desta etapa, você terá um ambiente local preparado para executar testes de upload com diferentes níveis de concorrência.

1. Configure a AWS CLI da forma abaixo:

```bash
aws configure set default.s3.max_concurrent_requests 1
aws configure set default.s3.multipart_threshold 64MB
aws configure set default.s3.multipart_chunksize 16MB
```

<details>
<summary><b>💡 Clique para entender: comandos de configuração da AWS CLI</b></summary>
<blockquote>

- `default.s3.max_concurrent_requests`: controla quantas solicitações podem acontecer em paralelo
- `default.s3.multipart_threshold`: define a partir de que tamanho o upload multipart será usado
- `default.s3.multipart_chunksize`: define o tamanho das partes em um upload multipart

Essas configurações afetam principalmente o comportamento da AWS CLI em uploads e downloads de arquivos grandes.

</blockquote>
</details>

2. Crie a pasta de trabalho e entre nela:

```bash
cd /workspaces/
mkdir s3-performance
cd s3-performance
export bucket=$(aws s3 ls | awk '/base-config-/ {print $3; exit}') && echo $bucket
```

---

## Parte 3 - Testando upload de arquivo grande

### Resultado esperado desta parte

Ao final desta etapa, você terá comparado o impacto da concorrência no upload de um arquivo grande.

1. Crie um arquivo de 5 GB para testar upload de arquivos grandes. Esse comando pode demorar um pouco:

```bash
dd if=/dev/zero of=5GB.file count=5120 bs=1M
```

2. Envie o arquivo para o bucket:

```bash
time aws s3 cp 5GB.file s3://${bucket}/upload1.test
```

3. Aumente a concorrência para 2 e teste o upload do mesmo arquivo:

```bash
aws configure set default.s3.max_concurrent_requests 2
time aws s3 cp 5GB.file s3://${bucket}/upload2.test
```

4. Aumente a concorrência para 10 e teste novamente:

```bash
aws configure set default.s3.max_concurrent_requests 10
time aws s3 cp 5GB.file s3://${bucket}/upload3.test
```

5. Retorne a concorrência para 1 antes de seguir:

```bash
aws configure set default.s3.max_concurrent_requests 1
```

### O que observar

- o tempo total do upload com concorrência 1
- o tempo total com concorrência 2
- o tempo total com concorrência 10

> [!IMPORTANT]
> O ganho de performance não costuma crescer de forma linear. O upload com concorrência 10 tende a ser melhor que com 2, mas não necessariamente 5 vezes mais rápido.

---

## Parte 4 - Upload paralelo de arquivos grandes

### Resultado esperado desta parte

Ao final desta etapa, você terá enviado vários arquivos grandes em paralelo para o S3.

1. Crie um arquivo de 1 GB para enviar 5 arquivos paralelamente:

```bash
dd if=/dev/zero of=1GB.file count=1024 bs=1M
```

2. Instale o pacote `parallel`:

```bash
sudo apt update -y && sudo apt-get install parallel -y
```

3. Envie os arquivos em paralelo:

```bash
time seq 1 5 | parallel --will-cite -j 5 aws s3 cp 1GB.file s3://${bucket}/parallel/object{}.test
```

<details>
<summary><b>💡 Clique para entender: upload com comando parallel</b></summary>
<blockquote>

Nesse comando:

- `seq 1 5` gera os números de 1 a 5
- `parallel -j 5` executa até 5 jobs simultaneamente
- `aws s3 cp` envia o mesmo arquivo para caminhos diferentes no bucket

Na prática, você cria 5 uploads independentes e paralelos.

</blockquote>
</details>

4. Delete o arquivo local de 1 GB:

```bash
rm 1GB.file
```

---

## Parte 5 - Performance com arquivos menores

### Resultado esperado desta parte

Ao final desta etapa, você terá comparado estratégias de envio para muitos arquivos menores.

1. Crie 2000 arquivos de 1 MB:

```bash
cd /workspaces/s3-performance
mkdir sync
seq -w 1 2000 | xargs -n1 -I% sh -c 'dd if=/dev/zero of=sync/file.% bs=1M count=1'
```

2. Defina a concorrência como 1 e envie os arquivos com `sync`:

```bash
aws configure set default.s3.max_concurrent_requests 1
export bucket=$(aws s3 ls | awk '/base-config-/ {print $3; exit}') && echo $bucket
time aws s3 sync sync/ s3://${bucket}/sync1/
```

<details>
<summary><b>💡 Clique para entender: upload com comando sync</b></summary>
<blockquote>

O comando `aws s3 sync` sincroniza conteúdo entre um diretório local e um bucket S3.

Ele:

- compara origem e destino
- copia apenas o que está faltando ou mudou
- pode ser usado para backup, sincronização e migração de dados

</blockquote>
</details>

3. Aumente a concorrência para 10 e execute novamente:

```bash
aws configure set default.s3.max_concurrent_requests 10
time aws s3 sync sync/ s3://${bucket}/sync2/
```

4. Delete os arquivos criados:

```bash
rm -rf sync
```

---

## Parte 6 - Testando arquivos muito pequenos

### Resultado esperado desta parte

Ao final desta etapa, você terá observado o impacto do overhead no envio de muitos arquivos minúsculos.

1. Crie os identificadores e o arquivo de 1 KB:

```bash
seq 1 500 > object_ids
cat object_ids
dd if=/dev/zero of=1KB.file count=1 bs=1K
aws configure set default.s3.max_concurrent_requests 1
```

2. Envie os arquivos para o bucket:

```bash
time parallel --will-cite -a object_ids -j 5 aws s3 cp 1KB.file s3://${bucket}/run1/{}
```

<details>
<summary><b>💡 Clique para entender: upload com comando parallel</b></summary>
<blockquote>

Aqui o GNU Parallel lê os IDs do arquivo `object_ids` e substitui `{}` no caminho de destino, criando múltiplos objetos distintos no S3.

Esse padrão é útil quando você quer testar ou executar uploads paralelos de muitos arquivos pequenos.

</blockquote>
</details>

---

## Parte 7 - Comparando opções de cópia no S3

### Resultado esperado desta parte

Ao final desta etapa, você terá comparado 3 formas diferentes de copiar um arquivo no S3.

1. Execute os comandos abaixo e compare o tempo de execução de cada abordagem:

```bash
aws configure set default.s3.max_concurrent_requests 1
time (aws s3 cp s3://$bucket/upload1.test 5GB.file; aws s3 cp 5GB.file s3://$bucket/copy/5GB.file)
time aws s3api copy-object --copy-source $bucket/upload1.test --bucket $bucket --key copy/5GB-2.file
time aws s3 cp s3://$bucket/upload1.test s3://$bucket/copy/5GB-3.file
```

<details>
<summary><b>💡 Clique para entender: comandos de cópia de arquivos no S3</b></summary>
<blockquote>

Você está comparando 3 abordagens:

1. `aws s3 cp` com download local e reupload
2. `aws s3api copy-object` com cópia server-side
3. `aws s3 cp` entre objetos S3, que pode usar cópia server-side

### O que observar

- tempo total
- necessidade ou não de transferir dados para a máquina local
- simplicidade operacional

### Tendência esperada

Em geral, as cópias server-side no próprio S3 são mais eficientes do que baixar e reenviar o arquivo.

</blockquote>
</details>

---

## Parte 8 - Limpeza do ambiente

### Resultado esperado desta parte

Ao final desta etapa, os arquivos criados no S3 e localmente terão sido removidos.

1. Delete os arquivos que você criou no bucket e localmente:

```bash
aws s3 rm s3://${bucket}/ --recursive
cd /workspaces
rm -rf s3-performance
```

---

## Conclusão

Se você chegou até aqui, você testou:

- uploads simples com `aws s3 cp`
- envios em lote com `aws s3 sync`
- uploads paralelos com `parallel`
- impacto da concorrência na AWS CLI
- diferença entre cópia local e cópia server-side

Esse laboratório serve como base para entender como decisões de transferência afetam desempenho, custo e experiência operacional no Amazon S3.
