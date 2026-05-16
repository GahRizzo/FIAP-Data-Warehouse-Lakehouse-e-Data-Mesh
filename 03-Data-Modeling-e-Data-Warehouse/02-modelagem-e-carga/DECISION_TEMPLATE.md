# Decisão de modelagem — Receita AUTOMOBILE 1995 por região

> Documento entregue à diretora financeira **Marina** da TPCH Trading antes da reunião do conselho. Simula um ADR (Architecture Decision Record) real — documento curto que registra uma escolha técnica com contexto, alternativas e consequências.
>
> **Como usar**: copie este arquivo para `DECISION.md` na mesma pasta e preencha enquanto avança no Lab 03.2. Os campos `N₁`, `N₂`, `N₃` são preenchidos ao final de cada Parte 2, 3, 4. A justificativa final é escrita na Parte 5.

---

## Autor e data

- **Engenheiro de dados**: _______ (você)
- **Stakeholder**: Marina (CFO, TPCH Trading)
- **RM**: _______
- **Data**: AAAA-MM-DD

---

## Contexto

Descreva em 3-5 linhas a pergunta de negócio que Marina te trouxe e o que se sabe sobre o dado disponível.

_(exemplo: "Marina pediu a receita líquida do segmento AUTOMOBILE em 1995 por região para a apresentação do conselho. Os dados vêm do ERP onde reclassificações de segmento acontecem esporadicamente. A `customer_history` registra 75k clientes que mudaram de segmento entre 1996 e 1998 — depois do recorte da pergunta.")_

---

## Três números observados

Preencha os resultados da query-âncora (receita AUTOMOBILE 1995, região AMERICA) nas três modelagens:

| Modelagem | Receita AUTOMOBILE 1995 (AMERICA) |
|-----------|-----------------------------------|
| A — Espelho OLTP | `N₁` = _______ |
| B — Star SCD1 | `N₂` = _______ |
| C — Star SCD2 | `N₃` = _______ |

**Diferença percentual entre a maior e a menor**: _______%

---

## Decisão

Marque uma opção e escreva por quê:

- [ ] Modelagem **A** (Espelho OLTP)
- [ ] Modelagem **B** (Star SCD1)
- [ ] Modelagem **C** (Star SCD2)

**Justificativa** (3-5 linhas):

_______

---

## Alternativas consideradas e descartadas

Para cada opção que você **não** escolheu, escreva uma linha dizendo por quê.

- **Opção X não escolhida**: _______
- **Opção Y não escolhida**: _______

---

## Consequências

### Positivas

- _______
- _______

### Negativas / pontos de atenção

- _______
- _______

---

## O que eu precisaria do negócio para validar esta escolha

Liste 2-3 perguntas que você faria para o stakeholder (CMO, diretor de operações, analista de receita) **antes** de colocar essa modelagem em produção.

1. _______
2. _______
3. _______

---

## Decisões técnicas secundárias

### Distribuição física da fato

- `DISTKEY` escolhida: _______
- `SORTKEY` escolhida: _______
- Por quê: _______

### Cálculo de receita

- Materializado em coluna (`vr_receita_liquida`) **ou** calculado em view / on-the-fly?
- Escolha: _______
- Por quê: _______

---

## Observações adicionais

_(espaço livre — comportamentos inesperados, descobertas, dúvidas que você quer anotar)_

_______
