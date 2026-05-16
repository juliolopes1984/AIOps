## Prompt
```
# Task
Analise as informações no arquivo Questão 03.csv anexo, referentes aos custos da AWS do último mês e retorne as oportunidades de redução de custos priorizadas por impacto informando o percentual de redução de cada recurso e ao final o percentual total de redução, informe o nível de esforço de implementação de cada etapa (baixo, médio ou alto), os risco e pré-requisitos

# Action
Liste todas as oportunidades de redução de custo ordenando da maior oportunidade para a menor em formato de tabela e um plano de execução por semana da primeira semana até o final do trimestre

# Goal
Objetivo: reduzir em, no mínimo, 15% o custo com cloud sem degradar o SLA e a performance até o final do próximo trimestre

```
---

## Modelo
```
Gemini 3 Flash e GPT-5.5
```
---

## Output Gemini 3 Flash

Com base nos dados do arquivo **Questão 03.csv**, sua fatura mensal atual totaliza **US$ 41.800,00**. Para atingir a meta de redução de, no mínimo, 15% (cerca de **US$ 6.270,00/mês**) sem comprometer o SLA, foquei em otimizações que atacam o baixo uso e os modelos de compra ineficientes.

Aqui está a análise técnica e o plano de ação estratégico.

---

### 📊 Oportunidades de Redução de Custos (Priorizadas por Impacto)

Abaixo, as oportunidades ordenadas do maior para o menor impacto financeiro estimado:

| Recurso / Serviço | Oportunidade de Otimização | Redução Estimada (%) | Economia Mensal (USD) | Esforço | Risco | Pré-requisitos |
| --- | --- | --- | --- | --- | --- | --- |
| **EC2 on-demand** | Migração para Savings Plans (1 ano) e Rightsizing (uso em 45%) | 35% | $2.870 | Baixo | Baixo | Análise de estabilidade do workload. |
| **RDS PostgreSQL** | Compra de Instância Reservada (RI) e Rightsizing (uso em 62%) | 30% | $2.460 | Baixo | Baixo | Previsibilidade de uso por 12 meses. |
| **CloudWatch Logs** | Redução de retenção (90 p/ 30 dias) e filtros de ingestão | 40% | $1.120 | Baixo | Baixo | Política de conformidade de logs da empresa. |
| **EKS (Clusters)** | Rightsizing de Nodes e uso de Spot Instances em Dev/Stage | 15% | $1.005 | Médio | Médio | Nodes stateless e tolerância a interrupções. |
| **S3 Standard** | Implementação de Lifecycle Policies (Intelligent Tiering) | 20% | $620 | Baixo | Mínimo | Classificação de dados (quentes vs. frios). |
| **NAT Gateway** | Implementação de VPC Endpoints (Gateway type p/ S3/Dynamo) | 15% | $180 | Médio | Baixo | Mapeamento de tráfego interno para AWS Services. |
| **Total Estimado** | **Mix de Otimizações** | **~19,7%** | **$8.255** | -- | -- | -- |

> **Nota:** A redução total estimada de **19,7%** supera a meta de 15%, garantindo uma margem de segurança caso algum workload não suporte instâncias Spot ou Rightsizing agressivo.

---

### 📅 Plano de Execução (Próximo Trimestre)

Para não sobrecarregar o time de engenharia e garantir o SLA, o plano é dividido em fases de "Ganhos Rápidos" (Quick Wins) até a "Otimização Estrutural".

#### **Mês 1: Quick Wins & Governança (Semanas 1 - 4)**

* **Semana 1:** Auditoria e alteração da retenção do **CloudWatch Logs** de 90 para 30 dias (ou conforme compliance).
* **Semana 2:** Ativação do **S3 Intelligent Tiering** em todos os buckets principais para mover dados frios automaticamente.
* **Semana 3:** Compra de **Compute Savings Plans** para cobrir o baseline do EC2 on-demand e Lambda.
* **Semana 4:** Compra de **Reserved Instances** para o RDS PostgreSQL (após confirmar que não haverá mudança de engine).

#### **Mês 2: Rightsizing e Eficiência de Rede (Semanas 5 - 8)**

* **Semana 5:** Análise detalhada de instâncias **EC2 on-demand** com uso abaixo de 45% para downgrade de tipo (ex: de `m5.large` para `m5.medium`).
* **Semana 6:** Aplicação do Rightsizing nas instâncias EC2 validadas.
* **Semana 7:** Implementação de **VPC Endpoints** para S3, reduzindo o tráfego processado pelo NAT Gateway.
* **Semana 8:** Revisão do **EBS gp3**. Embora o uso esteja em 68%, verificar volumes órfãos ou snapshots antigos para deleção.

#### **Mês 3: Modernização & Spot (Semanas 9 - 12)**

* **Semana 9:** Teste de **EKS Spot Instances** em ambiente de homologação/desenvolvimento.
* **Semana 10:** Implementação de Nodes Spot no cluster de produção do **EKS** para workloads não-críticos (workers, filas).
* **Semana 11:** Ajuste final de escalonamento (HPA/VPA) no EKS para otimizar o uso dos nodes existentes.
* **Semana 12:** Consolidação de resultados, fechamento do relatório de economia e setup de alertas de orçamento (AWS Budgets) para evitar retrocessos.

---

### ⚠️ Riscos e Considerações Finais

1. **Comprometimento Financeiro:** A compra de Savings Plans e RIs reduz o custo, mas "trava" o pagamento por 1 ou 3 anos. Certifique-se de que a arquitetura não mudará drasticamente nesse período.
2. **SLA de Disponibilidade:** No EKS, a migração para Spot deve ser feita com cuidado; sempre mantenha uma porcentagem de instâncias *On-Demand* para garantir que o serviço não caia totalmente em caso de falta de capacidade Spot na AWS.
3. **Logs:** Antes de reduzir a retenção do CloudWatch, verifique se o time de segurança ou auditoria não exige os 90 dias originais. Se sim, mova os logs para S3 Glacier (muito mais barato) em vez de apenas deletá-los.

Com essas ações, você não apenas bate a meta de 15%, como estabelece uma cultura de **FinOps** mais madura para o resto do ano!

---
## Output GPT-5.5
Analisei o arquivo e há oportunidades claras de otimização — principalmente em **EC2 on-demand, EKS, observabilidade e banco/cache subutilizados**. Abaixo está a priorização por impacto estimado, com percentuais realistas (baseados em boas práticas AWS) sem comprometer SLA.

---

# 📊 Oportunidades de Redução de Custos (priorizadas)

| #  | Recurso            | Custo Atual (USD) | Ação Recomendada                             | Redução Estimada | Economia (USD) | Esforço | Risco | Pré-requisitos                      |
| -- | ------------------ | ----------------- | -------------------------------------------- | ---------------- | -------------- | ------- | ----- | ----------------------------------- |
| 1  | EC2 on-demand      | 8200              | Migrar para Savings Plans + Rightsizing      | **35%**          | 2870           | Médio   | Médio | Análise de uso + compromisso mínimo |
| 2  | EKS                | 6700              | Rightsizing de nodes + autoscaling + spot    | **30%**          | 2010           | Médio   | Médio | Configurar Cluster Autoscaler       |
| 3  | RDS PostgreSQL     | 8200              | Rightsizing + Reserved Instances             | **25%**          | 2050           | Médio   | Baixo | Métricas de CPU/memória             |
| 4  | CloudWatch Logs    | 2800              | Reduzir retenção (90 → 30 dias) + compressão | **40%**          | 1120           | Baixo   | Baixo | Política de retenção definida       |
| 5  | ElastiCache Redis  | 2100              | Rightsizing (uso 40%)                        | **30%**          | 630            | Baixo   | Baixo | Monitorar latência                  |
| 6  | NAT Gateway        | 1200              | Substituir por NAT Instance ou VPC endpoints | **40%**          | 480            | Médio   | Médio | Revisão de arquitetura              |
| 7  | S3 Standard        | 3100              | Lifecycle → IA / Glacier                     | **30%**          | 930            | Baixo   | Baixo | Política de acesso aos dados        |
| 8  | Data Transfer      | 1900              | Reduzir tráfego cross-region + caching       | **20%**          | 380            | Médio   | Médio | Mapear fluxos                       |
| 9  | EBS gp3            | 1600              | Limpeza de volumes + rightsizing             | **20%**          | 320            | Baixo   | Baixo | Auditoria de volumes                |
| 10 | EC2 reservada      | 4200              | Otimizar instâncias (72% uso)                | **10%**          | 420            | Baixo   | Baixo | Rightsizing                         |
| 11 | Lambda             | 900               | Ajuste memória/timeout                       | **20%**          | 180            | Baixo   | Baixo | Monitoramento                       |
| 12 | CloudWatch Metrics | 900               | Reduzir métricas customizadas                | **15%**          | 135            | Baixo   | Baixo | Revisão de dashboards               |

---

# 💰 Resultado Consolidado

* **Custo total atual:** $41.700
* **Economia potencial:** $11.525
* **Redução total estimada:** **~27,6%**

✅ **Meta de 15% atingida com folga**

---

# 🧠 Principais Insights

* **Maior desperdício:** EC2 on-demand com apenas 45% de uso
* **Infra subutilizada:** Redis (40%) e Lambda (30%)
* **Observabilidade cara:** Logs com retenção excessiva
* **Arquitetura cara:** NAT Gateway e tráfego entre regiões

---

# 🗺️ Plano de Execução (por semana até o trimestre)

## 🔹 Semana 1–2 (ganhos rápidos – baixo risco)

* Reduzir retenção do CloudWatch Logs
* Implementar lifecycle no S3
* Ajustar Lambda (memória/timeout)
* Limpar volumes EBS não utilizados

👉 **Impacto esperado:** ~8–10%

---

## 🔹 Semana 3–5

* Rightsizing ElastiCache
* Rightsizing EC2 reservadas
* Reduzir métricas CloudWatch

👉 **Impacto acumulado:** ~12–15%

---

## 🔹 Semana 6–8

* Rightsizing RDS + compra de Reserved Instances
* Implementar Savings Plans para EC2

👉 **Impacto acumulado:** ~20–23%

---

## 🔹 Semana 9–10

* Otimizar cluster EKS (autoscaling + spot)

👉 **Impacto acumulado:** ~25%

---

## 🔹 Semana 11–12 (mais complexas)

* Substituir NAT Gateway
* Reduzir Data Transfer (arquitetura/cache/CDN)

👉 **Impacto final:** ~27–28%

---

# ⚠️ Riscos principais

* **Over-optimization (risco de performance)** → mitigar com monitoramento
* **Compromissos financeiros (Savings Plans / RI)** → analisar baseline antes
* **Mudanças em rede (NAT / transfer)** → podem afetar conectividade

---

# ✅ Conclusão

Você tem um cenário muito típico de cloud com:

* superdimensionamento,
* baixa eficiência de compra,
* e pouca governança de dados/logs.

Com execução disciplinada, dá para atingir **15% já no primeiro mês** e ultrapassar **25% até o fim do trimestre**, sem impacto em SLA.

---

Se quiser, posso montar um **plano técnico detalhado (com comandos AWS / Terraform)** para implementar cada etapa.

---

## Justificativa
```
A Task estebelece a análise dos dados do CSV com os valores do ambiente na AWS para mapeamento das oportunidades de economia, riscos e esforços; a Action especifica uma tabela técnica ordenada por impacto e um plano de execução semanal; a o Goal determina o objetivo final da tarefa que é a redução de 15% dos custos com Cloud até o final do trimestre sem comprometer o SLA e a performance.
```
