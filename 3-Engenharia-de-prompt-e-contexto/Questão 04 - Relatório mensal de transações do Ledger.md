## Prompt:
```
# Task
Analise o anexo Questão 04.sql onde está descrita a estrutura das duas tabelas mais relevantes do Ledger (PostgreSQL)

# Action
Escreva uma query SQL que faça a seguinte busca: 
categorias em produção no momento: subscription, one_time, refund e credit_adjustment; considere para o relatório somente as linhas com status completed; o campo amount_cents está em centavos de real, a saída deve apresentar o valor em reais com duas casas decimais e utilizando , como separador; a saída deve agrupar por meses, no formato YYYY-MM e por categoria, com duas métricas por linha: quantidade de transações e volume total em reais; oredene primeiro por mês de forma crescente e depois por categoria de forma crescente.

# Goal
Objetivo: busque as informações dos últimos 6 meses corridos a contar da data de 2026-04-24.

```
---

## Modelo:
```
Gemini 3.1 Pro e Opus 4.7
```
---

## Output Gemini 3.1 Pro:
```
Aqui está a query SQL (PostgreSQL) construída para atender a todos os requisitos solicitados:

sql
SELECT 
    TO_CHAR(created_at, 'YYYY-MM') AS mes,
    category AS categoria,
    COUNT(id) AS quantidade_transacoes,
    REPLACE(TO_CHAR(SUM(amount_cents) / 100.0, 'FM9999999999999990.00'), '.', ',') AS volume_total_reais
FROM 
    transactions
WHERE 
    status = 'completed'
    AND category IN ('subscription', 'one_time', 'refund', 'credit_adjustment')
    -- Últimos 6 meses corridos a contar de 2026-04-24 (inclusive)
    AND created_at >= '2026-04-24'::date - INTERVAL '6 months'
    AND created_at < '2026-04-24'::date + INTERVAL '1 day'
GROUP BY 
    TO_CHAR(created_at, 'YYYY-MM'),
    category
ORDER BY 
    mes ASC,
    categoria ASC;


### Explicação dos recursos utilizados:

* **Filtros de Data (Range Seguro):** Utilize `INTERVAL '6 months'` subtraído da data limite e `< data + INTERVAL '1 day'` para garantir que capturamos as transações até o último segundo do dia 24 de abril de 2026. Fazer isso dessa forma permite que o PostgreSQL faça o uso adequado do índice `idx_transactions_created_at`.
* **Conversão para Reais (`/ 100.0`):** Ao dividir por `100.0` (com o `.0`), forçamos o banco de dados a tratar o resultado como um valor decimal e não arredondar para o inteiro mais próximo.
* **Formatação de Moeda com Vírgula:** O PostgreSQL usa ponto `.` como separador decimal padrão. O uso do `TO_CHAR(..., 'FM...0.00')` força a exibição de exatas duas casas decimais, enquanto a função `REPLACE()` troca o ponto por vírgula no resultado final.
* **Agrupamento:** O `GROUP BY` considera a string do mês formatado em `YYYY-MM` e a respectiva categoria, alinhando com a exigência de exibir a saída de forma segregada por essas duas dimensões.
```

## Output Opus 4.7:
```
Olhando para o esquema e os requisitos, vou montar a query passo a passo.

Pontos importantes a considerar:

- **Janela de tempo**: "últimos 6 meses corridos" a contar de 2026-04-24 → de 2025-10-24 (inclusive) até 2026-04-24 (exclusivo, para não pegar o próprio dia 24/04 parcialmente, mas isso é interpretável — usarei `>=` e `<` para segurança).
- **Filtros**: `status = 'completed'` e `category IN ('subscription','one_time','refund','credit_adjustment')`.
- **Conversão centavos → reais**: dividir por 100.0 e formatar com vírgula como separador decimal usando `to_char` com locale-independente: máscara `'FM999G999G990D00'` + `to_char` aceita modificadores de agrupamento, mas a forma mais previsível é usar `TO_CHAR(valor, 'FM999999999990D00')` que já produz `,` como separador decimal no PostgreSQL quando `lc_numeric` está em pt_BR — porém isso depende do locale. A forma robusta é construir manualmente com `REPLACE`.
- **Agrupamento por mês**: `TO_CHAR(created_at, 'YYYY-MM')`.

Vou usar `REPLACE` para garantir a vírgula independente do locale do servidor:

sql
---
SELECT
    TO_CHAR(created_at, 'YYYY-MM')                                      AS mes,
    category                                                            AS categoria,
    COUNT(*)                                                            AS qtd_transacoes,
    REPLACE(
        TO_CHAR(SUM(amount_cents) / 100.0, 'FM999999999990.00'),
        '.', ','
    )                                                                   AS volume_reais
FROM transactions
WHERE status = 'completed'
  AND category IN ('subscription', 'one_time', 'refund', 'credit_adjustment')
  AND created_at >= DATE '2026-04-24' - INTERVAL '6 months'
  AND created_at <  DATE '2026-04-24'
GROUP BY
    TO_CHAR(created_at, 'YYYY-MM'),
    category
ORDER BY
    mes ASC,
    categoria ASC;
---

Observações sobre decisões que tomei:

A janela `>= 2025-10-24` e `< 2026-04-24` cobre exatamente 6 meses corridos terminando no dia da referência. Se a intenção for incluir o próprio dia 24/04/2026, basta trocar o segundo limite para `<= DATE '2026-04-24' + INTERVAL '1 day'` ou usar `<= '2026-04-24 23:59:59.999999'`. Como o enunciado diz "a contar da data de 2026-04-24", interpretei como ponto de corte (não incluído), o que é o comportamento mais comum em relatórios.

O `REPLACE(TO_CHAR(..., 'FM999999999990.00'), '.', ',')` força a vírgula como separador decimal independentemente do `lc_numeric` do servidor — mais previsível que confiar no locale. O `FM` remove espaços de preenchimento à esquerda. Se você também quiser separador de milhar com ponto (padrão BR, ex.: `1.234,56`), troque para:

sql
REPLACE(REPLACE(TO_CHAR(SUM(amount_cents) / 100.0, 'FM999G999G990D00'), '.', '#'), ',', '.')

mas isso depende novamente do locale; a versão acima (sem separador de milhar) é a mais segura sem mais informações.

O `ORDER BY mes, categoria` usa os aliases — funciona em PostgreSQL na cláusula `ORDER BY`. Como `mes` é texto no formato `YYYY-MM`, a ordenação lexicográfica coincide com a cronológica, então está correto.

Os índices existentes em `created_at`, `status` e `category` ajudam o planner a filtrar antes do `GROUP BY`. Se essa query for executada com frequência, um índice composto `(status, category, created_at)` seria mais eficiente que os três separados, mas isso é otimização fora do escopo do pedido.
```
---

## Justificativa:
```
A Task traz um contexto inicial de solicitação de análise da estrutura das principais tabelas do Ledger (PostgreSQL); a Action detalha a instrução de criação da query SQL passando as regras de formtação, agrupamento, ordenação e os filtros a serem aplicados; o Goal estabele o período o qual a query deve utilizar para as buscas das informações no banco.
```
