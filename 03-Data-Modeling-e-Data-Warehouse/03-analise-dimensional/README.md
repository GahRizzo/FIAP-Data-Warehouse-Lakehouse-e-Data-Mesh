# 03.3 - Evolução do negócio: quando a modelagem tem que mudar

> **6 meses depois do Lab 03.2.**
>
> A recomendação que mandamos para Marina passou no conselho. O DW da TPCH Trading entrou em produção. Por 4 meses, ninguém liga para o engenheiro de dados — sinal verde. Aí, num único trimestre, **três pessoas batem na sua porta**:
>
> - **Marina (CFO)**: *"Janeiro: começamos a vender em marketplaces. Eles cobram comissão variável por fornecedor. Quero que a receita líquida que eu reporto reflita isso. Mas e os relatórios que já publiquei? Vão mudar de número?"*
>
> - **Lucas (CMO)**: *"Março: definição antiga de 'cliente ativo' está obsoleta. Quero ranking mensal: ativo é quem comprou nos últimos 12 meses **e** tem saldo aceitável. Funciona?"*
>
> - **CEO**: *"Maio: o dashboard executivo que vocês fizeram demora 20 segundos para abrir. Inaceitável. SLA de 5 segundos até sexta-feira."*
>
> Três demandas, três tipos de pressão (financeira, comercial, executiva), três decisões de modelagem diferentes. Vamos atacar uma de cada vez — **na ordem em que chegaram** — porque é como acontece na vida real: você nunca decide tudo de uma vez, você reage a uma coisa, sente a consequência, e aí encara a próxima.

Este laboratório é o que acontece nesse trimestre. Vamos sentir na prática por que **modelagem raramente sobrevive inalterada de um trimestre para o outro**. Cada evolução de negócio é aplicada sobre o star schema do Lab 03.2, e cada uma força uma decisão de redesign.

> [!WARNING]
> **Pré-requisitos obrigatórios antes de começar:**
>
> - [ ] Credenciais AWS do Academy atualizadas no Codespaces — ver [Preparando Credenciais](../../00-create-codespaces/Inicio-de-aula.md)
> - [ ] Cluster Redshift `dw-aula3-<short_id>` em status `available` (Lab [03.1 · Provisionamento](../01-provisionamento/README.md) executado)
> - [ ] Schema `dw_star` populado com o star schema SCD Tipo 1 (Lab [03.1 · Parte 3](../02-modelagem-e-carga/README.md#parte-3---modelagem-b-star-schema-com-scd-tipo-1) inteira executada)
> - [ ] Você consegue conectar no Query Editor v2 ou via psql no Codespaces
>
> **Valide rapidamente rodando esta query no Query Editor v2 antes de prosseguir:**
>
> ```sql
> SELECT COUNT(*) AS linhas_fato FROM dw_star.f_vendas;
> -- Esperado: 59986052
> ```
>
> Se retornar `relation "dw_star.f_vendas" does not exist` ou `0 linhas`, volte ao Lab 03.2 e complete a Parte 3 antes de seguir aqui.

## O que você vai fazer

3 evoluções de negócio aplicadas sobre o `dw_star` existente, sem recarregar dados do zero. Tempo estimado: **70–90 min** em cluster `ra3.large` × 2 nós (execução pura ~5 min + tempo para você ler os blocos `<details>`, observar resultados das queries-alvo e refletir nas perguntas discursivas).

- **Evolução 1** — Nova fórmula de receita com comissão de marketplace (Views + Materialized Views versionadas)
- **Evolução 2** — Redefinição de "cliente ativo" (SCD Tipo 2 × fato snapshot periódico)
- **Evolução 3** — SLA de 5s no dashboard executivo (redesign de DISTKEY × MV pré-agregada)
- **Reflexão final** — 4 perguntas discursivas para fechar o raciocínio
- **Destruição da infra** — `terraform destroy` obrigatório ao final

## Arquitetura

![Arquitetura das três evoluções](img/arquitetura-03-2.png)

O diagrama resume: partindo do `dw_star` consolidado no Lab 03.2 (coluna da esquerda), três evoluções de negócio chegam no mesmo trimestre. Cada linha mostra a sequência **dor de negócio → solução técnica → decisão pedagógica**: Evolução 1 acrescenta comissão na receita (views + MV versionadas), Evolução 2 redefine "cliente ativo" (SCD2 vs. fato snapshot periódico), Evolução 3 exige SLA de 5s (redesign de distkey vs. MV pré-agregada). A faixa final sintetiza as quatro perguntas da reflexão do lab.

Fonte editável: [`img/arquitetura-03-2.drawio`](img/arquitetura-03-2.drawio).

## Principais pontos de aprendizagem

- versionar fórmulas de negócio com views lado a lado
- quando usar Materialized View vs. view regular
- escolher entre SCD Tipo 2 e fato snapshot periódico
- medir `EXPLAIN ANALYZE` antes de otimizar (sem chute!)
- redistribuição vs. pré-agregação como estratégias alternativas
- ler `STL_QUERY` para tempos reais de execução

## O que você terá ao final

Ao final deste laboratório, você terá aplicado três evoluções de negócio reais sobre o warehouse, comparado estratégias de implementação e documentado os trade-offs na prática. Terminará com uma reflexão escrita sobre **"recalcular histórico ou congelar?"** — uma das perguntas centrais da maturidade de um engenheiro de dados.

> [!TIP]
> Sempre que encontrar um bloco com o título **💡 Clique para entender**, abra esse trecho. Ele traz explicação detalhada do comando, contexto prático da aula e links oficiais para aprofundamento.

## Mapa do lab

| Parte | O que você faz | Passos | Tempo |
|-------|----------------|--------|-------|
| [Parte 1](#parte-1---preparação-e-validação-do-ambiente) | Valida o cluster + schema do Lab 03.2 | [1](#passo-1) · [2](#passo-2) | ~5 min |
| [Parte 2](#parte-2---evolução-1-nova-fórmula-de-receita) | Evolução 1 — nova fórmula de receita (v1 vs v2) | [3](#passo-3) · [4](#passo-4) · [5](#passo-5) · [6](#passo-6) · [7](#passo-7) · [8](#passo-8) · [9](#passo-9) · [10](#passo-10) | ~20 min |
| [Parte 3](#parte-3---evolução-2-redefinição-de-cliente-ativo) | Evolução 2 — redefinir "cliente ativo" (SCD2 vs snapshot) | [11](#passo-11) · [12](#passo-12) · [13](#passo-13) · [14](#passo-14) · [15](#passo-15) · [16](#passo-16) · [17](#passo-17) · [18](#passo-18) · [19](#passo-19) | ~25 min |
| [Parte 4](#parte-4---evolução-3-sla-de-5s-no-dashboard-executivo) | Evolução 3 — SLA de 5s (EXPLAIN + MV + redesign) | [20](#passo-20) · [21](#passo-21) · [22](#passo-22) · [23](#passo-23) · [24](#passo-24) · [25](#passo-25) · [26](#passo-26) · [27](#passo-27) · [28](#passo-28) | ~25 min |
| [Parte 5](#parte-5---reflexão-final) | Reflexão final escrita | (4 perguntas discursivas) | ~10 min |
| [Parte 6](#parte-6---destruindo-a-infraestrutura-ao-final-da-aula) | `terraform destroy` | (sem passos numerados) | ~8 min |

> [!TIP]
> Se travou em algum passo, você pode pular direto: clique no número do passo na coluna **Passos** acima.

---

## Contexto

A linha do tempo do trimestre, com Marina, Lucas e o CEO entrando em cena cada um na sua hora:

```mermaid
timeline
    title Trimestre pos-go-live do DW (T+4 meses)
    Janeiro  : Marina (CFO)<br/>Comissao de marketplace<br/>"Receita liquida muda de formula. E o historico que ja publiquei?"
    Marco    : Lucas (CMO)<br/>Redefinir 'cliente ativo'<br/>"Comprou nos ultimos 12m E saldo OK. Funciona?"
    Maio     : CEO<br/>SLA 5s no dashboard<br/>"Demora 20s. Inaceitavel. Quero 5s ate sexta."
```

Três pessoas, três meses diferentes, três tipos de pressão (financeira → comercial → executiva). **Resolvemos uma de cada vez, na ordem em que chegaram** — porque é como acontece na vida real: você reage a uma demanda, sente a consequência, e aí encara a próxima.

Cada evolução **muda o significado dos números** ou **muda como o dashboard performa**. E cada uma força uma decisão arquitetural diferente:

| Evolução | Tipo de mudança | Decisão central |
|----------|-----------------|-----------------|
| **1 — Marina (jan)** | Métrica que muda de fórmula | Recalcular histórico OU congelar a versão antiga? |
| **2 — Lucas (mar)** | Atributo dimensional que evolui no tempo | SCD2 OU fato snapshot periódico? |
| **3 — CEO (mai)** | Performance do consumo | Redesign físico OU pré-agregar? |

Vamos atacar uma evolução de cada vez, na ordem em que chegaram.

---

## Parte 1 - Preparação e validação do ambiente

### Resultado esperado desta parte

Ao final desta etapa, você estará conectado ao Redshift com `dw_star` acessível, e terá confirmado que os objetos do Lab 03.2 continuam disponíveis.

---

<a id="passo-1"></a>

1. No [Redshift Query Editor v2](https://us-east-1.console.aws.amazon.com/sqlworkbench/home?region=us-east-1), confirme que o schema `dw_star` existe e tem as tabelas esperadas. Usamos `SVV_TABLES` (system view oficial do Redshift) em vez de `pg_tables` porque a system view é confiável em qualquer modo de sessão:

```sql
SELECT table_schema, table_name, table_type
FROM   SVV_TABLES
WHERE  table_schema = 'dw_star'
ORDER  BY table_name;
```

O resultado deve conter as 6 linhas: `dim_customer`, `dim_data`, `dim_geografia`, `dim_produto`, `dim_supplier`, `f_vendas`.

![](img/dw_star_tables_ok.png)

---

<a id="passo-2"></a>

2. Confirme que `f_vendas` tem o volume correto:

```sql
SELECT COUNT(*) AS linhas FROM dw_star.f_vendas;
```

Esperado: **59.986.052** linhas.

### Checkpoint

Se você chegou até aqui, então o ambiente do Lab 03.2 está preservado e podemos começar as evoluções.

---

## Parte 2 - Evolução 1: nova fórmula de receita

> **Janeiro. Marina manda um e-mail**: *"Os marketplaces começaram a cobrar comissão variável (3% a 12%) por fornecedor. A receita líquida que eu reporto **precisa** descontar isso a partir de agora. Mas tenho um problema: já publiquei 6 trimestres de relatório com a fórmula antiga. Se eu mudar retroativamente, perco a comparabilidade. Como você resolve?"*
>
> Esta evolução **não é técnica, é semântica**: a fórmula da métrica mudou. Vamos discutir juntos a estratégia de versionar fórmulas (v1 e v2 coexistindo) em vez de sobrescrever.

### Por que essa evolução existe

| Aspecto | Resposta curta |
|---------|----------------|
| **Problema de negócio** | Métrica financeira (receita líquida) muda de fórmula. Decisão sensível: recalcular histórico OU congelar versão antiga? Bater meta de 1996, contrato de bônus, comparativo público — tudo depende do número antigo. |
| **Pergunta que essa evolução responde** | *"Como exponho duas fórmulas simultaneamente sem quebrar nada do que já está em produção?"* |
| **Pergunta que essa evolução **não** resolve** | *"Qual fórmula está certa?"* — não há resposta universal. O lab te ensina a tornar essa decisão **explícita** e **documentada**, não a tomar a decisão por você. |
| **Quando acontece na vida real** | Toda vez que uma métrica de KPI corporativo precisa mudar — comissionamento, regra contábil, redefinição de cohort. É uma das fontes de conflito mais comuns entre engenharia e finanças. |

### Resultado esperado desta parte

Ao final desta parte, a `dim_supplier` terá uma coluna `pct_comissao`, existirão duas views de receita (v1 com a fórmula antiga, v2 com a nova), e uma Materialized View com `AUTO REFRESH` pré-agregará a v2 por ano × mês × região × segmento.

---

<a id="passo-3"></a>

3. Acrescente o atributo `pct_comissao` em `dim_supplier`. Para fins didáticos, a comissão é determinística (entre 3% e 12%) baseada em `s_suppkey`:

```sql
ALTER TABLE dw_star.dim_supplier
    ADD COLUMN pct_comissao DECIMAL(5,4) NOT NULL DEFAULT 0.0;

UPDATE dw_star.dim_supplier
SET pct_comissao = ROUND( (((s_suppkey % 10) + 3) * 1.0 / 100), 4 );

ANALYZE dw_star.dim_supplier;
```

---

<a id="passo-4"></a>

4. Valide a distribuição das comissões:

```sql
SELECT
    pct_comissao,
    COUNT(*) AS qtd_fornecedores
FROM dw_star.dim_supplier
GROUP BY pct_comissao
ORDER BY pct_comissao;
```


![](img/pct_comissao_distribuicao.png)

<details>
<summary><b>💡 Clique para entender: por que ALTER TABLE é seguro aqui</b></summary>
<blockquote>

Adicionar uma coluna nullable/com default em uma tabela dimensional pequena (`dim_supplier` tem 10k linhas) é uma operação barata e reversível. O Redshift não precisa reescrever a tabela inteira.

### Quando ALTER TABLE se torna caro

- Tabelas grandes (fatos de milhões de linhas)
- Mudança de tipo de coluna (ex: `VARCHAR(10)` → `VARCHAR(50)`)
- Adição de `NOT NULL` sem default em tabela já populada

### Boa prática

Para dimensões, `ALTER TABLE ADD COLUMN` é geralmente OK. Para fatos grandes, considere recriar a tabela via `CREATE TABLE AS SELECT` com a coluna nova já presente — é mais rápido em alguns casos.

Documentação oficial:
- [ALTER TABLE no Redshift](https://docs.aws.amazon.com/redshift/latest/dg/r_ALTER_TABLE.html)

</blockquote>
</details>

---

<a id="passo-5"></a>

5. Crie a view v1 (fórmula antiga preservada) e a view v2 (nova fórmula com comissão):

```sql
CREATE OR REPLACE VIEW dw_star.v_receita_liquida_v1 AS
SELECT
    f.data_sk,
    f.customer_sk,
    f.produto_sk,
    f.supplier_sk,
    f.geografia_sk,
    f.nr_pedido,
    f.nr_linha_pedido,
    f.qt_vendida,
    f.vl_receita_liquida       AS vl_receita_liquida,
    'v1'                       AS versao_formula
FROM dw_star.f_vendas f;

CREATE OR REPLACE VIEW dw_star.v_receita_liquida_v2 AS
SELECT
    f.data_sk,
    f.customer_sk,
    f.produto_sk,
    f.supplier_sk,
    f.geografia_sk,
    f.nr_pedido,
    f.nr_linha_pedido,
    f.qt_vendida,
    (f.vl_preco_estendido
       * (1 - f.vl_desconto_pct)
       * (1 - s.pct_comissao))   AS vl_receita_liquida,
    'v2'                         AS versao_formula
FROM dw_star.f_vendas     f
JOIN dw_star.dim_supplier s ON s.supplier_sk = f.supplier_sk;
```

---

<a id="passo-6"></a>

6. Compare os totais para ver o impacto da comissão. Cada linha tem um rótulo descritivo deixando explícito qual fórmula foi aplicada:

```sql
SELECT 'v1 - SEM comissao (formula antiga)'  AS serie,
       ROUND(SUM(vl_receita_liquida), 2)     AS valor
FROM dw_star.v_receita_liquida_v1
UNION ALL
SELECT 'v2 - COM comissao (formula nova)',
       ROUND(SUM(vl_receita_liquida), 2)
FROM dw_star.v_receita_liquida_v2
UNION ALL
SELECT 'diferenca (v2 - v1)',
       ROUND(
         (SELECT SUM(vl_receita_liquida) FROM dw_star.v_receita_liquida_v2)
       - (SELECT SUM(vl_receita_liquida) FROM dw_star.v_receita_liquida_v1)
       , 2);
```

<!-- PRINT SUGERIDO: img/v1_vs_v2_total.png
     Tres linhas: v1 (sem comissao), v2 (com comissao, menor), diferenca (negativa).
     Aluno enxerga imediatamente qual e a com comissao e o tamanho do impacto. -->
![](img/v1_vs_v2_total.png)

> [!NOTE]
> A linha **v2** é sempre **menor** que **v1** porque desconta comissão. A linha **diferenca** é o impacto absoluto (negativo) — corresponde à comissão média ponderada pela receita.

<details>
<summary><b>💡 Clique para entender: por que preservar a v1 em vez de substituir</b></summary>
<blockquote>

Três razões pragmáticas para manter a v1 mesmo depois da v2 existir:

1. **Dashboards antigos continuam funcionando** enquanto você comunica a mudança — nenhum gestor descobre métrica diferente sem aviso prévio.
2. **Auditoria**: se alguém perguntar "como este número foi calculado em 1997?", você mostra a v1 e a resposta é completa.
3. **Comparabilidade temporal**: se você recalcular o histórico retroativo com a v2, perde a comparação com os relatórios oficiais publicados na época.

### Boa prática: janela de coexistência

Defina uma janela de coexistência (ex: 1 trimestre) em que as duas versões ficam disponíveis. Depois desse prazo, a v1 pode ser deprecada ou ter os consumidores migrados para v2.

### Versionamento explícito

Incluir a coluna `versao_formula` na view é boa prática. Quando um consumidor salva o resultado em cache ou outro sistema, fica claro qual fórmula foi aplicada.

Documentação oficial:
- [Views no Redshift](https://docs.aws.amazon.com/redshift/latest/dg/r_Views.html)

</blockquote>
</details>

---

<a id="passo-7"></a>

7. Crie a Materialized View da v2 para consumo recorrente em dashboards:

```sql
CREATE MATERIALIZED VIEW dw_star.mv_receita_liquida_v2_mensal
AUTO REFRESH YES
AS
SELECT
    d.nr_ano,
    d.nr_mes,
    g.nm_regiao,
    c.sg_segmento,
    SUM(
        f.vl_preco_estendido
          * (1 - f.vl_desconto_pct)
          * (1 - s.pct_comissao)
    )                                 AS vl_receita_v2,
    COUNT(*)                          AS qt_itens
FROM dw_star.f_vendas      f
JOIN dw_star.dim_data      d ON d.data_sk      = f.data_sk
JOIN dw_star.dim_geografia g ON g.geografia_sk = f.geografia_sk
JOIN dw_star.dim_customer  c ON c.customer_sk  = f.customer_sk
JOIN dw_star.dim_supplier  s ON s.supplier_sk  = f.supplier_sk
GROUP BY d.nr_ano, d.nr_mes, g.nm_regiao, c.sg_segmento;

REFRESH MATERIALIZED VIEW dw_star.mv_receita_liquida_v2_mensal;
```

---

<a id="passo-8"></a>

8. Veja o status da MV:

```sql
SELECT
    schema_name,
    name,
    is_stale,
    autorefresh,
    state
FROM svv_mv_info
WHERE schema_name = 'dw_star'
  AND name        = 'mv_receita_liquida_v2_mensal';
```

<!-- PRINT SUGERIDO: img/mv_status_ok.png
     Linha mostrando is_stale=f, autorefresh=t, state=1 (refresh incremental
     habilitado — caso ideal). -->
![](img/mv_status_ok.png)

> [!TIP]
> Resultado esperado: `is_stale=f`, `autorefresh=t`, `state=1`.
>
> - **`is_stale=f`** → MV está sincronizada com `f_vendas`.
> - **`autorefresh=t`** → Redshift vai refrescar automaticamente quando a base mudar.
> - **`state=1`** (inteiro, não texto): MV pode fazer **refresh incremental** (só processa o delta). Outros valores possíveis: `0` = recomputed (refresh completo), `-1` = exige recompute total.

<details>
<summary><b>⚠ Se <code>is_stale = true</code> ou <code>autorefresh = false</code></b></summary>
<blockquote>

A MV pode ficar desatualizada se houve mudança nas tabelas base e o refresh automático ainda não rodou. Force um refresh manual:

```sql
REFRESH MATERIALIZED VIEW dw_star.mv_receita_liquida_v2_mensal;
```

Se `autorefresh = false` mesmo após o `CREATE`, a MV pode ter operadores que o Redshift considera incompatíveis com auto refresh (funções não-determinísticas, algumas window functions). Simplifique a query e recrie.

</blockquote>
</details>

<details>
<summary><b>💡 Clique para entender: Materialized Views no Redshift</b></summary>
<blockquote>

Materialized View (MV) armazena o resultado de uma query pré-computada. A diferença crítica em relação a uma view normal:

- **View**: recalcula a cada consulta. Sempre fresca, custo alto em joins complexos.
- **MV**: materializa o resultado. Consumo em milissegundos, mas pode ficar "stale" até o próximo `REFRESH`.

### AUTO REFRESH YES

O Redshift monitora as tabelas base. Quando detecta que os dados mudaram, dispara refresh automaticamente em momentos de baixa utilização do cluster. Isso elimina a necessidade de agendamento manual.

### Limitação

MVs com `AUTO REFRESH` não podem conter certos tipos de expressão (ex: funções não-determinísticas, algumas window functions). Se o `CREATE` falhar, simplifique a query.

### Quando MV vale a pena

- Dashboards consumidos muitas vezes por dia com o mesmo corte
- Agregações pesadas sobre fatos grandes
- Consultas que caberiam em "cache" semântico

### Quando MV não compensa

- Queries ad-hoc que mudam a cada vez
- Dados que mudam em alta frequência (overhead de refresh)
- Agregações que ocupariam mais storage que a tabela base

Documentação oficial:
- [Materialized views](https://docs.aws.amazon.com/redshift/latest/dg/materialized-view-overview.html)
- [AUTO REFRESH](https://docs.aws.amazon.com/redshift/latest/dg/materialized-view-refresh.html)

</blockquote>
</details>

---

<a id="passo-9"></a>

9. Meça o impacto da nova fórmula por ano (análise pedagógica):

```sql
WITH v1_ano AS (
    SELECT d.nr_ano, SUM(v.vl_receita_liquida) AS receita_v1
    FROM dw_star.v_receita_liquida_v1 v
    JOIN dw_star.dim_data             d ON d.data_sk = v.data_sk
    GROUP BY d.nr_ano
),
v2_ano AS (
    SELECT d.nr_ano, SUM(v.vl_receita_liquida) AS receita_v2
    FROM dw_star.v_receita_liquida_v2 v
    JOIN dw_star.dim_data             d ON d.data_sk = v.data_sk
    GROUP BY d.nr_ano
)
SELECT
    v1.nr_ano,
    ROUND(v1.receita_v1, 2)                                 AS receita_v1,
    ROUND(v2.receita_v2, 2)                                 AS receita_v2,
    ROUND(v1.receita_v1 - v2.receita_v2, 2)                 AS diferenca_abs,
    ROUND((v1.receita_v1 - v2.receita_v2) / v1.receita_v1 * 100, 2) AS diferenca_pct
FROM v1_ano v1
JOIN v2_ano v2 ON v2.nr_ano = v1.nr_ano
ORDER BY v1.nr_ano;
```

![](img/v1_v2_por_ano.png)

---

<a id="passo-10"></a>

10. **Pergunta**: *"Marina precisa do dashboard mensal por região × segmento de 1997. Quantas linhas a engine processa em cada caminho — view (recalcula tudo) vs. MV (pré-agregada)?"*. Em vez de ler 2 `EXPLAIN` em sala, rodamos uma única query que conta volumes:

```sql
SELECT 'caminho A - via view (recalcula joins)' AS caminho,
       (SELECT COUNT(*) FROM dw_star.f_vendas
          WHERE data_sk BETWEEN 19970101 AND 19971231)
       + (SELECT COUNT(*) FROM dw_star.dim_customer)
       + (SELECT COUNT(*) FROM dw_star.dim_geografia)
       + (SELECT COUNT(*) FROM dw_star.dim_data
            WHERE nr_ano = 1997)                    AS linhas_processadas,
       3                                            AS qtd_joins
UNION ALL
SELECT 'caminho B - via MV (pre-agregada)',
       (SELECT COUNT(*) FROM dw_star.mv_receita_liquida_v2_mensal
          WHERE nr_ano = 1997),
       0
ORDER BY linhas_processadas DESC;
```

![](img/explain_view_vs_mv.png)

> [!TIP]
> Esperado: o **caminho A** processa milhões de linhas (≈ 8,5M de `f_vendas` 1997 + 1,5M de `dim_customer` + 25 + 365) com **3 joins**. O **caminho B** processa **algumas centenas de linhas** (12 meses × 5 regiões × 5 segmentos = ~300) com **zero joins**. **Mesma resposta de negócio, ~30.000× menos trabalho.** É isso que paga a MV: você troca um pouco de espaço em disco e refresh em background pelo SLA de leitura.

### Reflexão: recalcular ou congelar?

A view v1 preserva os números antigos exatamente como foram publicados. A MV da v2 atualiza toda vez que os dados base mudam.

**Quando faz sentido recalcular histórico com regra nova**:
- A métrica não foi usada para bater meta externa
- O regulador não exige reprodutibilidade da versão publicada
- A equipe quer uma comparação temporal consistente

**Quando congelar é mais seguro**:
- A métrica foi reportada publicamente ou para conselho
- Contratos ou bônus dependem do número antigo
- Auditoria exige consultar o valor exato vigente na época

### Checkpoint

Se você chegou até aqui, então:

- `dim_supplier` tem `pct_comissao`
- as duas views coexistem
- a MV tem `AUTO REFRESH` habilitado e está `OK`
- você observou o impacto da nova fórmula ano a ano

---

## Parte 3 - Evolução 2: redefinição de "cliente ativo"

> **Março. Lucas (CMO) chama na sala**: *"O conceito 'cliente ativo' que vocês usam está errado para mim. Preciso de algo mais sofisticado: ativo é quem **comprou nos últimos 12 meses** E **tem saldo aceitável** (não está endividado). Eu quero ver isso evoluir mês a mês — quantos ativos eu tinha em janeiro? Em fevereiro? Em qualquer mês de 1997?"*
>
> Esta evolução tem uma decisão de modelagem clássica: **modelar `is_active` como atributo de uma dimensão SCD2** (variável no tempo) ou **modelar como fato snapshot periódico** (1 linha por cliente × mês). Cada uma brilha em perguntas diferentes. Vamos construir as duas e ver na prática.

### Por que essa evolução existe

| Aspecto | Resposta curta |
|---------|----------------|
| **Problema de negócio** | Atributo de cliente que **muda no tempo** (status ativo/inativo) e o time de marketing precisa **série temporal** desse atributo — não basta saber o estado atual. |
| **Pergunta que essa evolução responde** | *"Quando modelo um atributo dimensional volátil, uso SCD2 (atributo na dimensão) ou um fato snapshot (atributo no fato)?"* |
| **Decisão central** | SCD2 é melhor para *"qual era o status em uma data específica?"*. Fato snapshot é melhor para *"qual a evolução mês a mês?"*. **As duas existem porque resolvem problemas diferentes.** |
| **Quando acontece na vida real** | Status de cliente (ativo/churn), tier de fidelidade (bronze/prata/ouro), nível de risco de crédito, classificação ABC de produtos. Em fintech, em e-commerce, em telecom. |

### Resultado esperado desta parte

Ao final desta parte, você terá implementado o conceito "cliente ativo" de **duas formas diferentes** — como atributo SCD Tipo 2 em `dim_customer_v2` e como fato snapshot periódico `f_customer_status_mensal` — e comparado onde cada abordagem brilha ou sofre.

### A regra de negócio

```
cliente ativo = comprou nos últimos 12 meses E saldo > -5000
```

### Opção A — SCD Tipo 2 em dim_customer_v2

---

<a id="passo-11"></a>

11. Crie a dimensão versionada. Esta abordagem modela `is_active` como um **atributo** do cliente que varia com o tempo:

```sql
CREATE TABLE dw_star.dim_customer_v2 (
    customer_v2_sk   BIGINT        NOT NULL,
    c_custkey        BIGINT        NOT NULL,
    nm_cliente       VARCHAR(25)   NOT NULL,
    sg_segmento      VARCHAR(10)   NOT NULL,
    vl_saldo         DECIMAL(15,2) NOT NULL,
    n_nationkey      INTEGER       NOT NULL,
    is_active        BOOLEAN       NOT NULL,
    valid_from       DATE          NOT NULL,
    valid_to         DATE          NOT NULL,
    is_current       BOOLEAN       NOT NULL
)
DISTKEY (c_custkey)
SORTKEY (c_custkey, valid_from);
```

---

<a id="passo-12"></a>

12. Popule a dimensão calculando transições de status mês a mês e colapsando em intervalos:

```sql
INSERT INTO dw_star.dim_customer_v2
WITH meses AS (
    SELECT DISTINCT
        DATE_TRUNC('month', dt_completa) :: DATE AS mes_ref
    FROM dw_star.dim_data
    WHERE dt_completa BETWEEN DATE '1993-01-01' AND DATE '1998-12-31'
),
compras_mes AS (
    SELECT
        c.c_custkey,
        DATE_TRUNC('month', d.dt_completa) :: DATE AS mes_compra
    FROM dw_star.f_vendas     f
    JOIN dw_star.dim_data     d ON d.data_sk     = f.data_sk
    JOIN dw_star.dim_customer c ON c.customer_sk = f.customer_sk
    GROUP BY c.c_custkey, DATE_TRUNC('month', d.dt_completa)
),
clientes_meses AS (
    SELECT cust.c_custkey, cust.vl_saldo, m.mes_ref
    FROM dw_star.dim_customer cust
    CROSS JOIN meses m
),
status_mensal AS (
    SELECT
        cm.c_custkey,
        cm.mes_ref,
        CASE WHEN MAX(cmp.c_custkey) IS NOT NULL AND cm.vl_saldo > -5000
             THEN TRUE ELSE FALSE END AS is_active
    FROM clientes_meses cm
    LEFT JOIN compras_mes cmp
      ON cmp.c_custkey = cm.c_custkey
     AND cmp.mes_compra >= DATEADD(month, -12, cm.mes_ref)
     AND cmp.mes_compra <  cm.mes_ref
    GROUP BY cm.c_custkey, cm.mes_ref, cm.vl_saldo
),
marcos AS (
    SELECT
        c_custkey,
        mes_ref,
        is_active,
        LAG(is_active) OVER (PARTITION BY c_custkey ORDER BY mes_ref) AS prev_status
    FROM status_mensal
),
transicoes AS (
    SELECT c_custkey, mes_ref AS valid_from, is_active
    FROM marcos
    WHERE prev_status IS NULL OR prev_status <> is_active
),
intervalos AS (
    SELECT
        c_custkey,
        valid_from,
        COALESCE(
            DATEADD(day, -1,
                LEAD(valid_from) OVER (PARTITION BY c_custkey ORDER BY valid_from)
            ),
            DATE '9999-12-31'
        ) AS valid_to,
        is_active
    FROM transicoes
)
SELECT
    ROW_NUMBER() OVER (ORDER BY i.c_custkey, i.valid_from)  AS customer_v2_sk,
    i.c_custkey,
    c.nm_cliente,
    c.sg_segmento,
    c.vl_saldo,
    c.n_nationkey,
    i.is_active,
    i.valid_from,
    i.valid_to,
    CASE WHEN i.valid_to = DATE '9999-12-31' THEN TRUE ELSE FALSE END AS is_current
FROM intervalos i
JOIN dw_star.dim_customer c ON c.c_custkey = i.c_custkey;

ANALYZE dw_star.dim_customer_v2;
```

<details>
<summary><b>💡 Clique para entender: a lógica da CTE de transições</b></summary>
<blockquote>

A estratégia aqui é clássica em data warehousing: dado um status mensal cliente × mês, **colapsar períodos consecutivos com o mesmo status** em um único intervalo `[valid_from, valid_to]`.

### Passo 1: `status_mensal`

Para cada cliente × mês, calcula TRUE/FALSE de "ativo" baseado na janela de 12 meses anteriores.

### Passo 2: `marcos` com `LAG`

Usa a função `LAG()` para pegar o status do mês anterior. Se o status mudou (ou é o primeiro mês do cliente), marca como transição.

### Passo 3: `transicoes`

Filtra apenas os meses em que houve mudança de status — esses são os pontos de início de cada intervalo.

### Passo 4: `intervalos` com `LEAD`

Usa `LEAD()` para pegar a próxima transição e calcular `valid_to = próxima transição - 1 dia`. Se não há próxima, usa `9999-12-31`.

### Custo

Este `INSERT` roda em poucos segundos mesmo com 1,5M clientes × 72 meses (108M pares). A chave é que window functions (`LAG`, `LEAD`, `ROW_NUMBER`) rodam bem paralelizadas em Redshift — com 2 nós o paralelismo dobra.

</blockquote>
</details>

---

<a id="passo-13"></a>

13. Pergunta analítica clássica que explora a SCD2 — quantos clientes estavam ativos em 1996-06-01?

```sql
SELECT
    COUNT(*) AS clientes_ativos_19960601
FROM dw_star.dim_customer_v2
WHERE DATE '1996-06-01' BETWEEN valid_from AND valid_to
  AND is_active = TRUE;
```

<!-- PRINT SUGERIDO: img/scd2_ativos_19960601.png
     Número único mostrando clientes ativos naquela data específica. -->
![](img/scd2_ativos_19960601.png)

### Opção B — Fato snapshot periódico

---

<a id="passo-14"></a>

14. Agora a abordagem alternativa: modelar o mesmo conceito como **fato**, com uma linha por cliente × mês:

```sql
CREATE TABLE dw_star.f_customer_status_mensal (
    data_sk        INTEGER  NOT NULL,
    customer_sk    BIGINT   NOT NULL,
    is_active      BOOLEAN  NOT NULL,
    qt_pedidos_12m INTEGER  NOT NULL,
    vl_receita_12m DECIMAL(18,2) NOT NULL
)
DISTKEY (customer_sk)
SORTKEY (data_sk);
```

---

<a id="passo-15"></a>

15. Popule a fato snapshot. Em SF10 são **~108M linhas** (1,5M clientes × 72 meses), o que é pesado de uma vez só — então particionamos a carga **ano por ano** dentro de uma `stored procedure` com loop. O aluno cria a proc uma vez e executa um único `CALL` (~5-6 min total):

```sql
CREATE OR REPLACE PROCEDURE dw_star.sp_load_customer_status_mensal(
    ano_inicio INT,
    ano_fim    INT
)
AS $$
DECLARE
    ano INT;
BEGIN
    FOR ano IN ano_inicio..ano_fim LOOP
        RAISE INFO 'carregando ano %', ano;

        INSERT INTO dw_star.f_customer_status_mensal
        WITH meses AS (
            SELECT DISTINCT
                DATE_TRUNC('month', dt_completa) :: DATE                              AS mes_ref,
                CAST(TO_CHAR(DATE_TRUNC('month', dt_completa), 'YYYYMMDD') AS INTEGER) AS data_sk
            FROM dw_star.dim_data
            WHERE EXTRACT(YEAR FROM dt_completa) = ano
        ),
        clientes_com_compra AS (
            SELECT DISTINCT customer_sk FROM dw_star.f_vendas
        ),
        compras_mensais AS (
            SELECT
                f.customer_sk,
                DATE_TRUNC('month', d.dt_completa) :: DATE AS mes_compra,
                COUNT(DISTINCT f.nr_pedido)                AS qt_pedidos,
                SUM(f.vl_receita_liquida)                  AS vl_receita
            FROM dw_star.f_vendas f
            JOIN dw_star.dim_data d ON d.data_sk = f.data_sk
            WHERE EXTRACT(YEAR FROM d.dt_completa) IN (ano - 1, ano)
            GROUP BY f.customer_sk, DATE_TRUNC('month', d.dt_completa)
        ),
        agregado_12m AS (
            SELECT
                m.mes_ref, m.data_sk, c.customer_sk, c.vl_saldo,
                COALESCE(SUM(cm.qt_pedidos), 0) AS qt_pedidos,
                COALESCE(SUM(cm.vl_receita), 0) AS vl_receita
            FROM meses m
            CROSS JOIN dw_star.dim_customer c
            LEFT JOIN compras_mensais cm
              ON cm.customer_sk = c.customer_sk
             AND cm.mes_compra >= DATEADD(month, -12, m.mes_ref)
             AND cm.mes_compra <  m.mes_ref
            WHERE c.customer_sk IN (SELECT customer_sk FROM clientes_com_compra)
            GROUP BY m.mes_ref, m.data_sk, c.customer_sk, c.vl_saldo
        )
        SELECT
            data_sk, customer_sk,
            CASE WHEN qt_pedidos > 0 AND vl_saldo > -5000 THEN TRUE ELSE FALSE END AS is_active,
            qt_pedidos AS qt_pedidos_12m,
            vl_receita AS vl_receita_12m
        FROM agregado_12m;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

CALL dw_star.sp_load_customer_status_mensal(1993, 1998);
ANALYZE dw_star.f_customer_status_mensal;
```

![](img/procedure_call_years.png)

> [!TIP]
> No Query Editor v2, expanda **Messages** depois de rodar o `CALL` para acompanhar os `RAISE INFO 'carregando ano XXXX'` — você vê em tempo real qual ano está processando. Se quiser pular a etapa pesada (~5-6 min) e seguir, troque os argumentos do `CALL` para `(1996, 1998)` — apenas 3 anos. As queries 16-19 ainda funcionam, só não têm dados anteriores a 1996.

<details>
<summary><b>💡 Clique para entender: por que particionar a carga por ano</b></summary>
<blockquote>

A versão "tudo de uma vez" faria CROSS JOIN de **72 meses × 1,5M clientes = 108M pares** + LEFT JOIN com 60M linhas de `f_vendas` + GROUP BY. Em `ra3.large × 2` isso ultrapassa a memória de trabalho do cluster e causa derrame para disco — de 25 minutos para 25 minutos sem retornar nada (timeout silencioso do WLM).

### A solução clássica: particionar por dimensão temporal

Em vez de calcular a fato snapshot inteira de uma vez, calculamos **um ano de cada vez**. Cada ano tem 12 meses × 1,5M clientes = 18M pares — muito mais leve. O resultado final é idêntico (`UNION ALL` implícito via 6 `INSERT INTO ... SELECT` consecutivos).

### Por que isso é a forma certa em produção

Pipelines de carga de fatos snapshot reais sempre particionam por uma janela temporal — diária ou mensal — pelo mesmo motivo: **isolar o tamanho de cada batch** para caber na janela de manutenção e na memória do cluster.

### Trade-off

Ganhamos previsibilidade e controle, perdemos um pouco de eficiência de planejador (cada INSERT planeja sozinho). Para datasets pequenos, fazer tudo de uma vez é mais simples; para datasets reais, particionar é mandatório.

</blockquote>
</details>

> [!TIP]
> Se quiser pular esta etapa pesada (~6 min total) e seguir para os passos seguintes, pode reduzir para **só os anos 1996-1998** — basta rodar 3 INSERTs em vez de 6. As queries 16-19 abaixo ainda funcionam, só não têm dados anteriores a 1996.

---

<a id="passo-16"></a>

16. Mesma pergunta (ativos em 1996-06-01), agora na fato snapshot:

```sql
SELECT
    data_sk,
    SUM(CASE WHEN is_active THEN 1 ELSE 0 END) AS qtd_ativos
FROM dw_star.f_customer_status_mensal
WHERE data_sk = 19960601
GROUP BY data_sk;
```

> [!TIP]
> Os dois resultados (SCD2 + snapshot) devem ser iguais dentro de diferenças de borda. Se divergirem muito, a definição de "ativo" entre as duas implementações está inconsistente.

![](img/snapshot_ativos_19960601.png)

### Comparando as abordagens

---

<a id="passo-17"></a>

17. Rode uma pergunta que **favorece o snapshot** — evolução mensal de ativos em 3 anos. Convertendo `data_sk` (`YYYYMMDD` inteiro) para `YYYY-MM` deixa a leitura cronológica imediata:

```sql
-- Via snapshot: trivialmente rápido
SELECT
    TO_CHAR(TO_DATE(data_sk::TEXT, 'YYYYMMDD'), 'YYYY-MM') AS mes,
    SUM(CASE WHEN is_active THEN 1 ELSE 0 END)             AS qtd_ativos
FROM dw_star.f_customer_status_mensal
WHERE data_sk BETWEEN 19960101 AND 19981231
GROUP BY data_sk
ORDER BY data_sk;
```

---

<a id="passo-18"></a>

18. Rode uma pergunta que **favorece a SCD2** — ativos AUTOMOBILE por mês de 1997:

```sql
-- Via SCD2: traz segmento + status na mesma tabela
SELECT
    DATE_TRUNC('month', f.dt_envio) :: DATE AS mes,
    COUNT(DISTINCT cv.c_custkey)            AS qtd_ativos_automobile
FROM dw_star.f_vendas f
JOIN dw_star.dim_customer_v2 cv
  ON cv.c_custkey   = (SELECT c_custkey FROM dw_star.dim_customer WHERE customer_sk = f.customer_sk)
 AND f.dt_envio BETWEEN cv.valid_from AND cv.valid_to
WHERE cv.is_active   = TRUE
  AND cv.sg_segmento = 'AUTOMOBILE'
  AND f.dt_envio BETWEEN DATE '1997-01-01' AND DATE '1997-12-31'
GROUP BY DATE_TRUNC('month', f.dt_envio)
ORDER BY mes;
```

---

<a id="passo-19"></a>

19. Rode a pergunta onde o snapshot é imbatível — clientes ativos por pelo menos 24 meses no total:

```sql
SELECT COUNT(*) AS clientes_ativos_24m_ou_mais
FROM (
    SELECT customer_sk
    FROM dw_star.f_customer_status_mensal
    WHERE is_active
    GROUP BY customer_sk
    HAVING COUNT(*) >= 24
) t;
```

<!-- PRINT SUGERIDO: img/snapshot_vs_scd2_queries.png
     Resultados das 3 perguntas diferentes que mostram os pontos fortes de cada abordagem. -->
![](img/snapshot_vs_scd2_queries.png)

<details>
<summary><b>💡 Clique para entender: SCD2 vs. snapshot — como escolher</b></summary>
<blockquote>

A escolha **não é binária**. Em warehouses maduros, você acaba tendo as duas — uma para cada tipo de pergunta.

### SCD2 ganha quando

- A pergunta é sobre "estado de um cliente em um instante" (filtrar vendas pelo segmento vigente na data)
- Precisa cruzar status com outros atributos dimensionais (segmento, região)
- A frequência de mudanças é baixa (é caro ter SCD2 em um atributo que muda diariamente)

### Snapshot periódico ganha quando

- A pergunta é sobre "quantos clientes em cada estado ao longo do tempo" (tendência, churn, engajamento)
- Precisa agregar contagens em múltiplos períodos
- A frequência de snapshot é natural (diária, semanal, mensal)

### O custo de ter as duas

- **Manutenção**: dois códigos de carga que podem divergir
- **Risco semântico**: se alguém usar a dimensão e outro usar a fato, os números podem não bater
- **Governança**: bom desenho força consumo por views/MVs que garantem consistência

### Regra prática

Começar com o que responde a pergunta dominante do trimestre. Promover a outra quando aparecer demanda explícita. **Nunca** implementar as duas "por via das dúvidas" sem consumidor claro.

</blockquote>
</details>

### Checkpoint

Se você chegou até aqui, então:

- `dim_customer_v2` está populada com status versionado
- `f_customer_status_mensal` tem ~10M linhas
- você observou cenários onde cada abordagem é mais natural

---

## Parte 4 - Evolução 3: SLA de 5s no dashboard executivo

> **Maio. Você está em uma reunião quando o CEO entra**: *"O dashboard que vocês fizeram para o board demora 20 segundos para abrir. Isso é constrangedor. Quero menos de 5 segundos. Você tem até sexta-feira."*
>
> Diferente das duas anteriores, esta evolução **não é sobre semântica** — os números estão certos. É sobre **performance**. Vamos primeiro **medir** o baseline com `EXPLAIN`, depois testar **duas estratégias diferentes** (redesign físico da fato × MV pré-agregada) e comparar lado a lado.

### Por que essa evolução existe

| Aspecto | Resposta curta |
|---------|----------------|
| **Problema de negócio** | Dashboard executivo recorrente com SLA explícito (5s). A query atual demora 20s. Não dá para mudar a fórmula nem o resultado — só o **caminho** que o cluster percorre para chegar nele. |
| **Pergunta que essa evolução responde** | *"Como reduzo tempo de query sem mudar nada do que ela retorna?"* — duas alavancas clássicas: (1) reorganizar **distribuição/sort** dos dados na fato, (2) **pré-agregar** o resultado em uma tabela menor. |
| **Decisão central** | Redesign físico beneficia **todo workload** que filtra por aquela dimensão (efeito amplo, mas demanda re-criar tabela grande). MV pré-agregada beneficia **só** o dashboard, mas é pontual e barata. |
| **Quando acontece na vida real** | Toda vez que um stakeholder C-level reclama de tempo de dashboard. É a evolução **mais frequente** em DWs maduros. |

### Resultado esperado desta parte

Ao final desta parte, você terá medido a query-alvo, testado **duas estratégias de otimização** (redesign físico da fato × MV pré-agregada) e documentado a decisão.

### A query-alvo

---

<a id="passo-20"></a>

20. Esta é a query que o dashboard executivo roda hoje. Execute **3 vezes seguidas** e anote o tempo mediano (a primeira execução costuma ser mais lenta por compilação de plano):

```sql
SELECT
    d.nr_ano,
    d.nr_mes,
    g.nm_regiao,
    c.sg_segmento,
    SUM(f.vl_receita_liquida) AS receita
FROM dw_star.f_vendas      f
JOIN dw_star.dim_data      d ON d.data_sk      = f.data_sk
JOIN dw_star.dim_geografia g ON g.geografia_sk = f.geografia_sk
JOIN dw_star.dim_customer  c ON c.customer_sk  = f.customer_sk
GROUP BY d.nr_ano, d.nr_mes, g.nm_regiao, c.sg_segmento
ORDER BY d.nr_ano, d.nr_mes, g.nm_regiao, c.sg_segmento;
```

> [!TIP]
> No Query Editor v2, o tempo de execução aparece no canto inferior do painel de resultados. Anote a mediana das 3 execuções. Esse é seu **baseline**.

![](img/baseline_execution_time.png)

<details>
<summary><b>⚠ Se os 3 tempos variam muito entre si</b></summary>
<blockquote>

A primeira execução inclui **compilação de plano** e **carregamento de blocos em cache**, então costuma ser bem mais lenta. Rode **3 vezes** e use a **mediana** — nunca confie em uma única medição. Se as 3 continuarem divergentes, espere 1 minuto (pode haver outra query em paralelo no cluster) e meça de novo.

</blockquote>
</details>

---

<a id="passo-21"></a>

21. Olhe o plano da query. Procure por `DS_DIST_KEY`, `DS_BCAST_INNER` ou `DS_DIST_INNER` — são indicadores de redistribuição:

```sql
EXPLAIN
SELECT
    d.nr_ano, d.nr_mes, g.nm_regiao, c.sg_segmento, SUM(f.vl_receita_liquida)
FROM dw_star.f_vendas      f
JOIN dw_star.dim_data      d ON d.data_sk      = f.data_sk
JOIN dw_star.dim_geografia g ON g.geografia_sk = f.geografia_sk
JOIN dw_star.dim_customer  c ON c.customer_sk  = f.customer_sk
GROUP BY d.nr_ano, d.nr_mes, g.nm_regiao, c.sg_segmento;
```

<details>
<summary><b>💡 Clique para entender: os marcadores DS_* do EXPLAIN</b></summary>
<blockquote>

O Redshift precisa colocar dados relacionados no mesmo slice para executar joins. Quando não consegue, marca a operação com um "Distribution Strategy":

- **`DS_DIST_NONE`**: dados já estão alinhados (situação ótima)
- **`DS_DIST_ALL_NONE`**: tabela com `DISTSTYLE ALL` sendo unida — também ótimo
- **`DS_DIST_INNER`**: redistribui a tabela interna (pequena) — OK
- **`DS_DIST_OUTER`**: redistribui a tabela externa (maior) — caro
- **`DS_BCAST_INNER`**: broadcast da tabela inteira para todos os slices — caro se a tabela for grande
- **`DS_DIST_BOTH`**: redistribui as duas tabelas — mais caro

### O que procurar em query lenta

Se a query-alvo redistribui `f_vendas` (6M linhas), vale mudar a `DISTKEY` da fato. Se broadcasta uma dimensão grande, vale mudar a diststyle dela.

Documentação oficial:
- [Analyzing the query plan](https://docs.aws.amazon.com/redshift/latest/dg/c-analyzing-the-query-plan.html)

</blockquote>
</details>

### Estratégia A — Redesign físico da fato

---

<a id="passo-22"></a>

22. Recrie `f_vendas` com `DISTKEY(data_sk)` e `SORTKEY` composta. A tabela antiga é preservada como backup:

```sql
ALTER TABLE dw_star.f_vendas RENAME TO f_vendas_original;

CREATE TABLE dw_star.f_vendas (
    data_sk              INTEGER       NOT NULL,
    customer_sk          BIGINT        NOT NULL,
    produto_sk           BIGINT        NOT NULL,
    supplier_sk          BIGINT        NOT NULL,
    geografia_sk         INTEGER       NOT NULL,
    nr_pedido            BIGINT        NOT NULL,
    nr_linha_pedido      INTEGER       NOT NULL,
    qt_vendida           DECIMAL(15,2) NOT NULL,
    vl_preco_estendido   DECIMAL(15,2) NOT NULL,
    vl_desconto_pct      DECIMAL(15,2) NOT NULL,
    vl_imposto_pct       DECIMAL(15,2) NOT NULL,
    vl_receita_bruta     DECIMAL(18,4) NOT NULL,
    vl_receita_liquida   DECIMAL(18,4) NOT NULL,
    vl_receita_final     DECIMAL(18,4) NOT NULL,
    fl_retornado         CHAR(1)       NOT NULL,
    fl_status_linha      CHAR(1)       NOT NULL,
    dt_envio             DATE          NOT NULL,
    dt_recebimento       DATE          NOT NULL
)
DISTKEY (data_sk)
COMPOUND SORTKEY (data_sk, geografia_sk);

INSERT INTO dw_star.f_vendas
SELECT * FROM dw_star.f_vendas_original;

ANALYZE dw_star.f_vendas;
```

> [!IMPORTANT]
> O `VACUUM` **precisa rodar em uma transação separada** — Redshift não aceita `VACUUM` no meio de um bloco multi-statement. Rode-o em um bloco próprio:

```sql
VACUUM dw_star.f_vendas;
```

<details>
<summary><b>⚠ Se o redesign demorar muito</b></summary>
<blockquote>

Recriar `f_vendas` com dados copiados envolve mover 60M linhas (SF10). Em `ra3.large` × 2 nós isso leva tipicamente **3 a 5 minutos** — é esperado, não é travamento. O `VACUUM` no final adiciona mais ~30 segundos.

Se passar de 5 minutos, abra outra aba e confira se o cluster não está bloqueado por outra query concorrente:

```sql
SELECT pid, starttime, TRIM(query) AS query
FROM stv_recents
WHERE status = 'Running';
```

</blockquote>
</details>

<details>
<summary><b>💡 Clique para entender: por que DISTKEY(data_sk) agora?</b></summary>
<blockquote>

No Lab 03.2, escolhemos `DISTKEY(customer_sk)` porque não sabíamos qual seria o workload dominante. Agora sabemos: o dashboard executivo **filtra e agrupa por data/região**.

### Por que DISTKEY(data_sk) ajuda

Colocar todas as vendas do mesmo mês no mesmo slice permite que o `GROUP BY nr_ano, nr_mes` aconteça localmente, sem redistribuir dados entre slices. A operação final é só `UNION ALL` dos resultados parciais.

### Trade-off

Isso **prejudica** queries que filtram por cliente específico (ex: "qual o extrato do cliente X?"), porque agora os dados de X estão espalhados. É um trade-off real.

### COMPOUND SORTKEY (data_sk, geografia_sk)

Em vez de uma única coluna de sort, usamos duas. Compound sortkey cria a ordenação por primeira coluna, depois pela segunda dentro de cada valor da primeira. Filtros por `data_sk` continuam com skipping perfeito, e filtros adicionais por `geografia_sk` ganham skipping parcial.

### Alternativa: INTERLEAVED SORTKEY

Para workload multi-dimensional onde nenhuma coluna domina, `INTERLEAVED` dá pesos iguais. Tem custo de manutenção maior (requer `VACUUM REINDEX`), então só vale em cenários específicos.

Documentação oficial:
- [Choosing sort keys](https://docs.aws.amazon.com/redshift/latest/dg/t_Sorting_data.html)
- [Distribution styles](https://docs.aws.amazon.com/redshift/latest/dg/t_Distributing_data.html)

</blockquote>
</details>

---

<a id="passo-23"></a>

23. Rode a query-alvo novamente (3x) com a fato remodelada e anote o novo tempo:

```sql
SELECT
    d.nr_ano, d.nr_mes, g.nm_regiao, c.sg_segmento, SUM(f.vl_receita_liquida) AS receita
FROM dw_star.f_vendas      f
JOIN dw_star.dim_data      d ON d.data_sk      = f.data_sk
JOIN dw_star.dim_geografia g ON g.geografia_sk = f.geografia_sk
JOIN dw_star.dim_customer  c ON c.customer_sk  = f.customer_sk
GROUP BY d.nr_ano, d.nr_mes, g.nm_regiao, c.sg_segmento
ORDER BY d.nr_ano, d.nr_mes, g.nm_regiao, c.sg_segmento;
```

<details>
<summary><b>⚠ Se a query continuar lenta mesmo após redistkey</b></summary>
<blockquote>

Duas causas mais comuns:

1. **Estatísticas desatualizadas**. Verifique que o `ANALYZE` do passo 22 rodou até o fim. Rode de novo:

   ```sql
   ANALYZE dw_star.f_vendas;
   ```

2. **Skew alto** (distribuição desigual entre slices). Meses com muito mais vendas que outros geram desbalanceamento:

   ```sql
   SELECT "schema", "table", diststyle, sortkey1, size, tbl_rows, skew_rows
   FROM svv_table_info
   WHERE "schema" = 'dw_star' AND "table" = 'f_vendas';
   ```

   `skew_rows` alto (acima de 1.5) indica desbalanceamento e explica por que a redistkey sozinha não resolveu. Considere partir para a **Estratégia B (MV pré-agregada)** abaixo.

</blockquote>
</details>

### Estratégia B — Materialized View pré-agregada

---

<a id="passo-24"></a>

24. Agora crie uma MV com o corte exato que o dashboard consome:

```sql
CREATE MATERIALIZED VIEW dw_star.mv_dashboard_executivo
AUTO REFRESH YES
AS
SELECT
    d.nr_ano,
    d.nr_mes,
    g.nm_regiao,
    c.sg_segmento,
    SUM(f.vl_receita_liquida)    AS receita_liquida,
    SUM(f.vl_receita_final)      AS receita_final,
    SUM(f.qt_vendida)            AS qt_vendida,
    COUNT(*)                     AS qt_itens,
    COUNT(DISTINCT f.customer_sk) AS qt_clientes
FROM dw_star.f_vendas      f
JOIN dw_star.dim_data      d ON d.data_sk      = f.data_sk
JOIN dw_star.dim_geografia g ON g.geografia_sk = f.geografia_sk
JOIN dw_star.dim_customer  c ON c.customer_sk  = f.customer_sk
GROUP BY d.nr_ano, d.nr_mes, g.nm_regiao, c.sg_segmento;

REFRESH MATERIALIZED VIEW dw_star.mv_dashboard_executivo;
```

---

<a id="passo-25"></a>

25. Agora o dashboard consulta a MV em vez da fato. Rode 3x e anote o tempo:

```sql
SELECT
    nr_ano,
    nr_mes,
    nm_regiao,
    sg_segmento,
    receita_liquida
FROM dw_star.mv_dashboard_executivo
ORDER BY nr_ano, nr_mes, nm_regiao, sg_segmento;
```

<!-- PRINT SUGERIDO: img/mv_dashboard_execution_time.png
     Tempo de execução via MV — deve ser ordens de grandeza menor que o baseline. -->
![](img/mv_dashboard_execution_time.png)

### Comparando as três medições

---

<a id="passo-26"></a>

26. Monte a tabela mental:

| Estratégia | Tempo mediano (s) | Custo de implantação | Manutenção |
|-----------|-------------------|----------------------|------------|
| Baseline (DISTKEY customer_sk) | _______ | zero (já estava assim) | zero |
| Estratégia A (DISTKEY data_sk) | _______ | alto (recriar fato) | baixa |
| Estratégia B (MV pré-agregada) | _______ | baixo (só CREATE MV) | média (refresh + storage) |

---

<a id="passo-27"></a>

27. Veja as últimas execuções de cada estratégia via system table:

```sql
SELECT
    CASE
        WHEN querytxt ILIKE '%mv_dashboard_executivo%' THEN 'MV pre-agregada'
        WHEN querytxt ILIKE '%dim_customer%' AND querytxt ILIKE '%dim_geografia%' THEN 'Fato'
        ELSE 'outra'
    END                                           AS estrategia,
    starttime,
    DATEDIFF(ms, starttime, endtime)              AS duracao_ms,
    SUBSTRING(querytxt, 1, 80)                    AS query_snippet
FROM stl_query
WHERE userid > 1
  AND starttime > DATEADD(hour, -1, GETDATE())
  AND querytxt NOT ILIKE '%EXPLAIN%'
  AND (
      querytxt ILIKE '%mv_dashboard_executivo%'
   OR (querytxt ILIKE '%dim_customer%' AND querytxt ILIKE '%dim_geografia%')
  )
ORDER BY starttime DESC
LIMIT 20;
```

<!-- PRINT SUGERIDO: img/stl_query_comparison.png
     Lista das últimas 20 execuções, com as duas estratégias lado a lado para comparação objetiva. -->
![](img/stl_query_comparison.png)

### Decisão recomendada

Tipicamente, a **MV pré-agregada vence por ordens de grandeza** para este caso. Ela:

- responde em milissegundos (é uma tabela pequena já calculada)
- não exige recriar 6M linhas da fato
- tem refresh automático sem intervenção humana

A mudança de DISTKEY beneficia **todo** o workload que filtre por data — não só o dashboard — mas também força re-teste de outras queries.

> [!IMPORTANT]
> Em produção, o padrão maduro combina as duas: DISTKEY adequada ao workload dominante + MVs dedicadas para top-5 dashboards.

### Rollback (opcional)

---

<a id="passo-28"></a>

28. Se quiser voltar ao estado original do Lab 03.2, faça:

```sql
DROP TABLE dw_star.f_vendas;
ALTER TABLE dw_star.f_vendas_original RENAME TO f_vendas;
ANALYZE dw_star.f_vendas;
```

### Checkpoint

Se você chegou até aqui, então:

- você mediu o baseline com `EXPLAIN` + 3 execuções
- testou a estratégia de redistkey
- testou a estratégia de MV pré-agregada
- decidiu com base em dados reais, não em regra de polegar

---

## Parte 5 - Reflexão final

As perguntas abaixo não têm resposta única, mas têm respostas **melhores** que outras. Use-as para fechar o lab discutindo em sala (ou anotando no caderno, ou estendendo o recado-Marina do Lab 03.2 com mais 4 linhas para Lucas/CEO):

**Pergunta R1.** **"Você recalcularia o histórico com a nova fórmula de receita, ou manteria o número antigo congelado?"**
Em que contexto a resposta muda?

**Pergunta R2.** **"O que é 'correto': o número da métrica no dia em que foi publicada, ou o número recalculado com a regra atual?"**
Existe resposta universal, ou depende de quem está pedindo?

**Pergunta R3.** **"Se o CEO pedir 'a mesma tabela de 2023', você entrega como ela era, ou como ela fica aplicando as regras de hoje aos dados antigos?"**
Como a modelagem ajuda a separar esses dois mundos?

**Pergunta R4.** **"Você começaria o próximo warehouse implementando SCD2 em tudo, ou SCD1 e só promoveria para SCD2 quando surgisse demanda explícita?"**
Existe "padrão seguro"?

> [!TIP]
> A maturidade aparece em responder com "depende de X e Y" em vez de "sempre Z". Este lab é um exercício de raciocínio, não de memorização.

---

## Parte 6 - Destruindo a infraestrutura ao final da aula

### Resultado esperado desta parte

Ao final desta etapa, todos os recursos AWS criados para o Lab 03 (cluster Redshift, bucket S3, Glue Database, Security Group) terão sido removidos da sua conta, zerando o consumo de budget.

> [!CAUTION]
> **Este passo é obrigatório ao final da aula.** Um cluster Redshift `ra3.large` continua consumindo budget mesmo ocioso, e o Learner Lab tem orçamento limitado — se ultrapassar, sua conta é desativada e **todo o progresso é perdido**.

### Quanto custa deixar a infra ligada

Preços públicos AWS para a região `us-east-1` (valores de referência on-demand, 2026):

| Recurso | Preço unitário | Custo/dia (24h) | Custo/mês (720h) |
|---------|---------------|-----------------|-------------------|
| **Redshift `ra3.large` × 2 nós** | ~$0,51/hora | **~$12,24** | **~$367** |
| **Redshift Managed Storage** (dados do TPC-H SF10) | $0,024/GB-mês | ~$0,30 | ~$8 |
| **S3 Standard** (~10 GB de `.tbl`) | $0,023/GB-mês | negligível | ~$0,23 |
| **Glue Data Catalog** (primeiros 1M objetos) | grátis | $0 | $0 |
| **Total estimado** | — | **~$12,50/dia** | **~$375/mês** |

> [!IMPORTANT]
> **O cluster Redshift é 99% do custo.** Um fim de semana esquecido (~72h) consome ~$37 do seu budget do Learner Lab. Uma semana inteira, ~$87. Sem destroy, em poucas semanas o budget acaba e a conta é desativada — perdendo todo o progresso.

### 28. Volte ao terminal do Codespaces

Se você estava no Redshift Query Editor v2 no browser, alterne para a aba/janela do **GitHub Codespaces**. No Codespaces, abra um terminal integrado (`Ctrl+`` ` ou menu `Terminal → New Terminal`).

<!-- PRINT SUGERIDO: img/codespaces_terminal_aberto.png
     Codespaces com um terminal aberto, mostrando o prompt pronto para executar comandos. -->
![](img/codespaces_terminal_aberto.png)

### 29. Acesse a pasta de provisionamento

A infra foi criada pelo Terraform localizado em `01-provisionamento/`. Entre nela:

```bash
cd /workspaces/FIAP-Data-Warehouse-Lakehouse-e-Data-Mesh/03-Data-Modeling-e-Data-Warehouse/01-provisionamento
```

Confirme que você está no diretório certo (deve haver `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`):

```bash
ls -la
```

### 30. Verifique o que está provisionado antes de destruir

Opcional, mas boa prática. Liste os outputs para confirmar quais recursos serão removidos:

```bash
cd /workspaces/FIAP-Data-Warehouse-Lakehouse-e-Data-Mesh/03-Data-Modeling-e-Data-Warehouse/01-provisionamento
terraform output
```

Você deve ver o `redshift_cluster_identifier`, `s3_bucket_name` e `glue_database_name`.

<!-- PRINT SUGERIDO: img/terraform_output_antes_destroy.png
     Terminal mostrando os outputs do Terraform, com os 3 recursos principais visíveis.
     É o último retrato do que existe antes do destroy. -->
![](img/terraform_output_antes_destroy.png)

### 31. Execute o destroy

```bash
cd /workspaces/FIAP-Data-Warehouse-Lakehouse-e-Data-Mesh/03-Data-Modeling-e-Data-Warehouse/01-provisionamento
terraform destroy -auto-approve
```

O Terraform vai listar os **15 recursos** que serão deletados e iniciar a remoção imediatamente (a flag `-auto-approve` pula o "type 'yes' to confirm").

> [!NOTE]
> O destroy leva tipicamente **3 a 5 minutos**. O cluster Redshift é o mais demorado de destruir (~3 minutos) porque o serviço precisa liberar storage, snapshots e ENIs associadas.

<details>
<summary><b>💡 Clique para entender: o que o terraform destroy faz por baixo dos panos</b></summary>
<blockquote>

O Terraform lê o `terraform.tfstate` local, identifica todos os recursos gerenciados, calcula o grafo de dependências e destrói **na ordem inversa** da criação.

### Ordem típica de deleção

1. **Objetos S3** (prefixos `.keep`) — primeiros porque não têm dependência
2. **Security Group Rules** (ingress e egress) — antes do SG
3. **Security Group** — depois que as rules foram removidas
4. **Bucket S3** — depois que está vazio (`force_destroy = true` garante isso)
5. **Glue Database** — independente, cai junto
6. **Redshift Cluster** — o mais demorado
7. **Redshift Subnet Group** — depois que o cluster saiu
8. **random_password** — último, porque só é usado pelo cluster

### Se der erro no meio do destroy

Rode `terraform destroy -auto-approve` novamente. O Terraform é **idempotente** — ele só tenta destruir o que ainda existe. Causas comuns:

- Cluster Redshift em estado `modifying`: aguarde 30s e tente de novo
- Dependência temporária não liberada (ex: ENI presa): espere 1-2 min e repita
- Credenciais AWS expiradas: renove e tente novamente

Documentação oficial:
- [terraform destroy](https://developer.hashicorp.com/terraform/cli/commands/destroy)

</blockquote>
</details>

### 32. Confirme que tudo foi removido

Rode os 3 comandos abaixo. Todos devem retornar **vazio**:

```bash
# Nenhum cluster começando com "dw-aula3"
aws redshift describe-clusters \
    --query 'Clusters[?starts_with(ClusterIdentifier, `dw-aula3`)].ClusterIdentifier' \
    --output text

# Nenhum bucket começando com "dw-lab"
aws s3 ls | grep dw-lab

# Nenhum Glue database começando com "tpch_raw_"
aws glue get-databases \
    --query 'DatabaseList[?starts_with(Name, `tpch_raw_`)].Name' \
    --output text
```

<!-- PRINT SUGERIDO: img/aws_cleanup_verificado.png
     Terminal mostrando os 3 comandos acima e retorno vazio em todos.
     Essa é a prova de que a limpeza funcionou. -->
![](img/aws_cleanup_verificado.png)

### 33. Feche o Codespaces

Finalize também a sessão de trabalho:

```bash
# Ainda no terminal do Codespaces
exit
```

Depois vá em [github.com/codespaces](https://github.com/codespaces), clique nos 3 pontinhos ao lado do ambiente e selecione **Stop Codespace**. Isso preserva suas horas gratuitas de Codespaces para a próxima aula.

### Checkpoint final

Se você chegou até aqui, então:

- o cluster Redshift foi destruído (deixa de gerar custo)
- o bucket S3 foi removido
- o Glue Database foi deletado
- o Security Group foi limpo
- o Codespaces foi parado

> [!TIP]
> Você pode recriar todo o ambiente em qualquer outra aula rodando `terraform apply -auto-approve` novamente — leva 5-8 minutos e reproduz exatamente o mesmo estado. Essa é a beleza de infra como código.

---

## Conclusão

Se você chegou até aqui, você aplicou:

- versionamento de fórmula de negócio com views v1/v2 coexistindo
- Materialized View com AUTO REFRESH
- SCD2 vs. fato snapshot periódico — duas modelagens para a mesma pergunta
- medição objetiva de performance com EXPLAIN + stl_query
- redesign físico da fato vs. MV dedicada como estratégias de otimização
- teardown completo da infraestrutura via `terraform destroy`

Este é o fechamento da trilha de Data Modeling + Data Warehouse. Você construiu um warehouse, sentiu as escolhas de modelagem impactarem números concretos, viu como a evolução do negócio força o modelo a se adaptar e aprendeu a limpar tudo para preservar o budget.

---

<details>
<summary><b>💡 Como pedir ajuda se travou</b></summary>
<blockquote>

Antes de abrir issue/perguntar no Slack, colete estas 4 informações — elas reduzem o tempo de resposta em 10×:

1. **Em que passo você está** (ex: "Evolução 3, rodando o EXPLAIN da MV")
2. **Mensagem de erro literal** (copia-cola completo, texto é pesquisável)
3. **Saída de `SELECT * FROM SYS_QUERY_HISTORY ORDER BY start_time DESC LIMIT 5`** (mostra se a query rodou, falhou, ou está em fila)
4. **O que você já tentou** (evita sugestão redundante)

Canais (em ordem de prioridade):

- **Issues do repositório**: [github.com/vamperst/FIAP-Data-Warehouse-Lakehouse-e-Data-Mesh/issues](https://github.com/vamperst/FIAP-Data-Warehouse-Lakehouse-e-Data-Mesh/issues)
- **E-mail do professor**: `rafael.barbosa@fiap.com.br`
- **Antes de tudo**: releia o `<details>` mais próximo com `⚠ Se der erro` — cobre a maioria dos casos conhecidos

</blockquote>
</details>
