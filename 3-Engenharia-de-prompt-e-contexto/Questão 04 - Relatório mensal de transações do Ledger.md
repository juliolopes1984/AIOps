## Prompt:
```
# Task
Analise o anexo a estrutura das tabelas transactions e customers, as mais relevantes do Ledger (PostgreSQL)

CREATE TABLE transactions (
  id              BIGSERIAL PRIMARY KEY,
  customer_id     BIGINT NOT NULL REFERENCES customers(id),
  category        VARCHAR(32) NOT NULL,
  amount_cents    BIGINT NOT NULL,
  status          VARCHAR(16) NOT NULL,
  payment_method  VARCHAR(16),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_transactions_created_at ON transactions(created_at);
CREATE INDEX idx_transactions_status ON transactions(status);
CREATE INDEX idx_transactions_category ON transactions(category);

CREATE TABLE customers (
  id          BIGSERIAL PRIMARY KEY,
  segment     VARCHAR(16) NOT NULL,
  country     CHAR(2) NOT NULL,
  signup_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

# Action
Escreva uma query SQL que faça a seguinte busca: 
categorias em produção no momento: subscription, one_time, refund e credit_adjustment; considere para o relatório somente as linhas com status completed; o campo amount_cents está em centavos de real, a saída deve apresentar o valor em reais com duas casas decimais e utilizando , como separador; a saída deve agrupar por meses, no formato YYYY-MM e por categoria, com duas métricas por linha: quantidade de transações e volume total em reais; oredene primeiro por mês de forma crescente e depois por categoria de forma crescente.

# Goal
Objetivo: busque as informações dos últimos 6 meses corridos a contar da data de 2026-04-24.

```
---

## Modelo:
```
Gemini 3.1 Pro e Opus 4.8
```
---

## Output Gemini 3.1 Pro:

Aqui está a query SQL para o PostgreSQL que atende a todos os seus requisitos.

Note que a tabela `customers` não precisou ser unida (`JOIN`) nesta consulta, pois todas as informações necessárias para este relatório (categoria, status, datas e valores) já estão contidas na tabela `transactions`.

### Query SQL

```sql
SELECT 
    TO_CHAR(created_at, 'YYYY-MM') AS mes,
    category AS categoria,
    COUNT(*) AS quantidade_transacoes,
    REPLACE(TO_CHAR(SUM(amount_cents) / 100.0, 'FM999999999990.00'), '.', ',') AS volume_total_reais
FROM 
    transactions
WHERE 
    status = 'completed'
    AND category IN ('subscription', 'one_time', 'refund', 'credit_adjustment')
    -- Últimos 6 meses corridos a contar de 2026-04-24 (2025-10-24 até o final do dia 2026-04-24)
    AND created_at >= '2026-04-24'::DATE - INTERVAL '6 months'
    AND created_at < '2026-04-25'::DATE
GROUP BY 
    TO_CHAR(created_at, 'YYYY-MM'),
    category
ORDER BY 
    mes ASC, 
    categoria ASC;

```

---

### Detalhamento das decisões técnicas:

* **Conversão de Moeda e Formatação:** O campo `amount_cents` é somado primeiro e, em seguida, dividido por `100.0` (o `.0` garante que o PostgreSQL trate o resultado como decimal e não arredonde por divisão inteira). A função `TO_CHAR` com o modificador `FM999999999990.00` força a saída com duas casas decimais fixas, e o `REPLACE` troca o ponto nativo do banco de dados pela vírgula (padrão brasileiro).
* **Filtro de Data (`created_at`):** Utilizei a regra matemática de datas nativa do PostgreSQL. `INTERVAL '6 months'` garante retroagir exatamente para o dia 24 de outubro. O operador `< '2026-04-25'` foi utilizado no lugar de `<=` para garantir que transações ocorridas até o último segundo de `2026-04-24 (23:59:59)` sejam contabilizadas adequadamente, respeitando o tipo `TIMESTAMPTZ`.
* **Agrupamento:** O `TO_CHAR` foi utilizado no campo de data diretamente no `GROUP BY` e no `SELECT` para padronizar a granularidade mensal (`YYYY-MM`), conforme solicitado.
* **Performance:** A query tira pleno proveito do índice existente `idx_transactions_created_at` (devido à busca por range de datas) e do índice `idx_transactions_status` ao buscar apenas as transações completadas.

---
## Output Opus 4.8:

Segue a query, com explicação das decisões logo abaixo.

```sql
SELECT
    to_char(date_trunc('month', t.created_at), 'YYYY-MM')                  AS mes,
    t.category                                                             AS categoria,
    count(*)                                                               AS qtd_transacoes,
    translate(
        to_char(sum(t.amount_cents) / 100.0, 'FM999,999,999,990.00'),
        '.,', ',.'
    )                                                                      AS volume_reais
FROM transactions AS t
WHERE t.status = 'completed'
  AND t.category IN ('subscription', 'one_time', 'refund', 'credit_adjustment')
  AND t.created_at >= DATE '2026-04-24' - INTERVAL '6 months'
  AND t.created_at <  DATE '2026-04-24'
GROUP BY
    date_trunc('month', t.created_at),
    t.category
ORDER BY
    date_trunc('month', t.created_at) ASC,
    t.category ASC;
```

Pontos de decisão e premissas (vale conferir se batem com o que você espera):

**Janela dos 6 meses.** Interpretei "6 meses corridos a contar de 2026-04-24" como uma janela móvel de 6 meses que termina nessa data: `created_at >= '2025-10-24'` e `created_at < '2026-04-24'`. Como `created_at` é um timestamp, usei `<` para a data final (em vez de `<=`) para não cortar transações do próprio dia 24 de forma ambígua. Note que, por ser janela móvel, o agrupamento por `YYYY-MM` vai render buckets parciais nas pontas (out/2025 a partir do dia 24, e abr/2026 até o dia 24). Se você quer os 6 meses-calendário *completos*, troque a faixa por `>= date_trunc('month', DATE '2026-04-24') - INTERVAL '6 months'` e `< date_trunc('month', DATE '2026-04-24')`.

**Qual data usar.** Usei `created_at` por ser `NOT NULL`, indexada e ser a data natural da transação — o filtro fica eficiente com `idx_transactions_created_at`. Se o relatório deve ancorar no momento em que a transação foi *concluída*, é só trocar os três usos de `created_at` por `completed_at` (que é anulável, então pode valer adicionar `AND t.completed_at IS NOT NULL`).

**Formatação em reais.** `amount_cents / 100.0` converte centavos para reais, e o `to_char` + `translate` produz o padrão brasileiro (ponto como separador de milhar, vírgula como decimal), ex.: `1.234.567,89`. Se você não quer o separador de milhar, troque o formato por `'FM999990.00'`. Valores negativos (caso refunds/ajustes sejam armazenados assim) saem com `-` na frente normalmente.

Quer que eu ajuste para meses-calendário completos ou para ancorar em `completed_at`?

---

## Justificativa:
```
A Task traz um contexto inicial de solicitação de análise da estrutura das principais tabelas do Ledger (PostgreSQL); a Action detalha a instrução de criação da query SQL passando as regras de formtação, agrupamento, ordenação e os filtros a serem aplicados; o Goal estabele o período o qual a query deve utilizar para as buscas das informações no banco.
```
