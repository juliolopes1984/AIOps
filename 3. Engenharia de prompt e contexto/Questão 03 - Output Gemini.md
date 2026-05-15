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