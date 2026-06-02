## Prompt
```
# Task
Analise as informações abaixo referentes aos custos da AWS do último mês e retorne as oportunidades de redução de custos priorizadas por impacto informando o percentual de redução de cada recurso e ao final o percentual total de redução, informe o nível de esforço de implementação de cada etapa (baixo, médio ou alto), os risco e pré-requisitos

servico,categoria,custo_mensal_usd,uso_medio_pct,observacao
EC2 reservada,compute,4200,72,contrato de 1 ano
EC2 on-demand,compute,8200,45,workloads variaveis
EKS,compute,6700,58,3 clusters
RDS PostgreSQL,databases,8200,62,multi-AZ
ElastiCache Redis,databases,2100,40,cluster de producao
S3 Standard,storage,3100,,5 buckets principais
EBS gp3,storage,1600,68,volumes de producao
CloudWatch Logs,observability,2800,,retencao de 90 dias
CloudWatch Metrics,observability,900,,
Data Transfer Out,network,1900,,trafego entre regioes
NAT Gateway,network,1200,,3 gateways ativos
Lambda,compute,900,30,~12M invocacoes/mes

# Action
Liste todas as oportunidades de redução de custo ordenando da maior oportunidade para a menor em formato de tabela e um plano de execução por semana da primeira semana até o final do trimestre

# Goal
Objetivo: reduzir em, no mínimo, 15% o custo com cloud sem degradar o SLA e a performance até o final do próximo trimestre

```
---

## Modelo
```
Gemini 3.5 Flash e GPT-5.5
```
---

## Output Gemini 3.5 Flash

```python
import pandas as pd

data = [
    ["EC2 reservada", "compute", 4200, 72, "contrato de 1 ano"],
    ["EC2 on-demand", "compute", 8200, 45, "workloads variaveis"],
    ["EKS", "compute", 6700, 58, "3 clusters"],
    ["RDS PostgreSQL", "databases", 8200, 62, "multi-AZ"],
    ["ElastiCache Redis", "databases", 2100, 40, "cluster de producao"],
    ["S3 Standard", "storage", 3100, None, "5 buckets principais"],
    ["EBS gp3", "storage", 1600, 68, "volumes de producao"],
    ["CloudWatch Logs", "observability", 2800, None, "retencao de 90 dias"],
    ["CloudWatch Metrics", "observability", 900, None, ""],
    ["Data Transfer Out", "network", 1900, None, "trafego entre regioes"],
    ["NAT Gateway", "network", 1200, None, "3 gateways ativos"],
    ["Lambda", "compute", 900, 30, "~12M invocacoes/mes"]
]

df = pd.DataFrame(data, columns=["servico", "categoria", "custo_mensal_usd", "uso_medio_pct", "observacao"])

total_cost = df['custo_mensal_usd'].sum()

# Define reduction opportunities
# 1. EC2 On-demand: Rightsizing (30% reduction) + Savings Plans (25% on remaining) = ~47% reduction
# 2. RDS: Reserved Instances (30% reduction) + Rightsizing (10%) = ~37% reduction
# 3. CloudWatch Logs: Reduce retention and filter (35%)
# 4. S3: Lifecycle Policies/Intelligent Tiering (20%)
# 5. NAT Gateway: VPC Endpoints (40%)
# 6. ElastiCache: Rightsizing (25%)
# 7. EKS: Spot Instances for non-prod + Node optimization (15%)

opportunities = [
    {"servico": "EC2 on-demand", "pct_reducao": 45, "esforço": "Médio", "risco": "Downtime planejado para alteração de tipo de instância", "pre_req": "Monitoramento de métricas de CPU/RAM por 2 semanas"},
    {"servico": "RDS PostgreSQL", "pct_reducao": 30, "esforço": "Baixo", "risco": "Comprometimento financeiro (Lock-in)", "pre_req": "Previsão de uso estável para os próximos 12 meses"},
    {"servico": "CloudWatch Logs", "pct_reducao": 40, "esforço": "Baixo", "risco": "Perda de logs históricos necessários para auditoria", "pre_req": "Revisão de políticas de compliance e retenção"},
    {"servico": "S3 Standard", "pct_reducao": 25, "esforço": "Baixo", "risco": "Custo de recuperação de dados se mal configurado", "pre_req": "Análise de padrões de acesso aos buckets"},
    {"servico": "NAT Gateway", "pct_reducao": 35, "esforço": "Médio", "risco": "Interrupção momentânea de tráfego de rede", "pre_req": "Mapeamento de destinos (S3, DynamoDB) para criar Endpoints"},
    {"servico": "ElastiCache Redis", "pct_reducao": 20, "esforço": "Médio", "risco": "Pressão de memória em picos de uso", "pre_req": "Validação de métricas de 'FreeableMemory'"},
    {"servico": "EKS", "pct_reducao": 15, "esforço": "Alto", "risco": "Instabilidade em pods se não houver tolerância a falhas", "pre_req": "Configuração de Pod Disruption Budgets e HPA"}
]

results = []
for opp in opportunities:
    cost = df[df['servico'] == opp['servico']]['custo_mensal_usd'].values[0]
    reduction_usd = cost * (opp['pct_reducao'] / 100)
    results.append({
        "Recurso": opp['servico'],
        "Redução %": f"{opp['pct_reducao']}%",
        "Economia Mensal (USD)": reduction_usd,
        "Esforço": opp['esforço'],
        "Risco": opp['risco'],
        "Pré-requisitos": opp['pre_req']
    })

results_df = pd.DataFrame(results).sort_values(by="Economia Mensal (USD)", ascending=False)
total_savings = results_df['Economia Mensal (USD)'].sum()
total_reduction_pct = (total_savings / total_cost) * 100

print(f"Total Cost: {total_cost}")
print(f"Total Savings: {total_savings}")
print(f"Total Reduction %: {total_reduction_pct}")
print(results_df.to_string())



```

```text
Total Cost: 41800
Total Savings: 9890.0
Total Reduction %: 23.66028708133971
             Recurso Redução %  Economia Mensal (USD) Esforço                                                    Risco                                              Pré-requisitos
0      EC2 on-demand       45%                 3690.0   Médio   Downtime planejado para alteração de tipo de instância          Monitoramento de métricas de CPU/RAM por 2 semanas
1     RDS PostgreSQL       30%                 2460.0   Baixo                     Comprometimento financeiro (Lock-in)           Previsão de uso estável para os próximos 12 meses
2    CloudWatch Logs       40%                 1120.0   Baixo      Perda de logs históricos necessários para auditoria               Revisão de políticas de compliance e retenção
6                EKS       15%                 1005.0    Alto  Instabilidade em pods se não houver tolerância a falhas                Configuração de Pod Disruption Budgets e HPA
3        S3 Standard       25%                  775.0   Baixo         Custo de recuperação de dados se mal configurado                    Análise de padrões de acesso aos buckets
4        NAT Gateway       35%                  420.0   Médio                Interrupção momentânea de tráfego de rede  Mapeamento de destinos (S3, DynamoDB) para criar Endpoints
5  ElastiCache Redis       20%                  420.0   Médio                       Pressão de memória em picos de uso                   Validação de métricas de 'FreeableMemory'


```

Com base na análise dos custos da AWS do último mês, identificamos um potencial de economia de aproximadamente **23,6% (US$ 9.890,00)**, superando a meta de 15%.

Abaixo estão as oportunidades priorizadas por impacto financeiro e o plano de execução trimestral.

### 1. Oportunidades de Redução de Custos Priorizadas

| Recurso | Redução % | Economia Mensal (USD) | Esforço | Riscos | Pré-requisitos |
| --- | --- | --- | --- | --- | --- |
| **EC2 On-demand** | 45% | $3.690 | Médio | Downtime para troca de tipo de instância. | Monitorar CPU/RAM por 14 dias (Rightsizing). |
| **RDS PostgreSQL** | 30% | $2.460 | Baixo | Lock-in financeiro com a AWS. | Plano de capacidade estável por 12 meses (Reserved Instance). |
| **CloudWatch Logs** | 40% | $1.120 | Baixo | Perda de histórico para auditorias. | Revisar políticas de retenção e compliance. |
| **EKS** | 15% | $1.005 | Alto | Instabilidade se o app não for resiliente. | Configurar HPA, VPA e Pod Disruption Budgets. |
| **S3 Standard** | 25% | $775 | Baixo | Custo de transição se houver muitos objetos pequenos. | Habilitar S3 Intelligent-Tiering nos buckets principais. |
| **NAT Gateway** | 35% | $420 | Médio | Erros transitórios de rede na migração. | Mapear tráfego para S3/DynamoDB (VPC Endpoints). |
| **ElastiCache Redis** | 20% | $420 | Médio | Pressão de memória em picos de carga. | Validar métricas de `FreeableMemory` e desalocação. |

**Redução Total Estimada: 23,6% (~US$ 9.890,00/mês)**

---

### 2. Plano de Execução Trimestral

#### **Mês 1: Quick Wins e Diagnóstico (Semanas 1-4)**

* **Semana 1:** Implementação de **S3 Intelligent-Tiering** em todos os buckets e revisão de políticas de ciclo de vida (S3 Glacier).
* **Semana 2:** Ajuste de retenção do **CloudWatch Logs** (ex: reduzir de 90 para 30 dias onde permitido) e exclusão de métricas não utilizadas.
* **Semana 3:** Início da coleta de métricas detalhadas (Rightsizing) para EC2 e RDS usando AWS Compute Optimizer.
* **Semana 4:** Auditoria de **Data Transfer Out** e identificação de fluxos entre zonas de disponibilidade que podem ser otimizados.

#### **Mês 2: Otimização de Recursos e Instâncias (Semanas 5-8)**

* **Semana 5:** Execução do **Rightsizing de EC2 On-demand**: Redimensionar instâncias com uso médio de 45% para tipos menores ou gerações mais recentes (ex: m5 para m6g/m7g).
* **Semana 6:** **Rightsizing de RDS**: Avaliar se a instância atual comporta um downgrade ou mudança para Graviton (maior performance/menor custo).
* **Semana 7:** Otimização do **ElastiCache**: Ajustar nós de Redis baseando-se na memória real utilizada (atualmente em 40%).
* **Semana 8:** Implementação de **VPC Endpoints** (Gateway e Interface) para reduzir o tráfego processado pelo NAT Gateway.

#### **Mês 3: Estrutural e Compromissos Financeiros (Semanas 9-12)**

* **Semana 9:** Modernização do **EKS**: Implementar Karpenter para autoscaling eficiente e migrar workloads de Dev/Staging para **Spot Instances**.
* **Semana 10:** Compra de **Savings Plans ou Reserved Instances** para o novo "baseline" de computação (EC2 e RDS) após os ajustes de rightsizing.
* **Semana 11:** Revisão de volumes **EBS gp3**: Ajustar IOPS e Throughput para o estritamente necessário (baseline de produção).
* **Semana 12:** Relatório final de impacto, verificação do SLA/Performance e definição de governança (Tags obrigatórias e orçamentos no AWS Budgets).

---

### Resumo dos Resultados Esperados

* **Custo Atual:** US$ 41.800,00
* **Custo Projetado (Final do Trimestre):** US$ 31.910,00
* **Impacto no SLA:** Nenhum. As reduções focam em desperdício de alocação (overprovisioning) e otimização de modelo de compra, mantendo a redundância Multi-AZ e a disponibilidade dos clusters.

---

## Output GPT-5.5

Com base nos dados fornecidos, o custo mensal atual é de **US$ 41.700/mês**. Para atingir a meta de **redução mínima de 15% sem impacto em SLA e performance**, o potencial identificado é de aproximadamente **US$ 8.240/mês (19,8%)**, o que supera a meta estabelecida.

## Oportunidades de redução priorizadas

| Prioridade | Serviço            | Ação Recomendada                                                    | Redução Estimada (%) | Economia Mensal (USD) | Esforço | Riscos                                          | Pré-requisitos                        |
| ---------- | ------------------ | ------------------------------------------------------------------- | -------------------- | --------------------- | ------- | ----------------------------------------------- | ------------------------------------- |
| 1          | EC2 On-Demand      | Migrar carga estável para Savings Plans e Rightsizing               | 30%                  | 2.460                 | Médio   | Subdimensionamento se análise inadequada        | 30 dias de métricas detalhadas        |
| 2          | CloudWatch Logs    | Reduzir retenção de 90 para 30 dias e exportar histórico para S3    | 50%                  | 1.400                 | Baixo   | Dificuldade em investigações muito antigas      | Aprovação de compliance               |
| 3          | ElastiCache Redis  | Rightsizing e revisão de nós superdimensionados                     | 35%                  | 735                   | Médio   | Possível aumento de latência em pico            | Testes de carga                       |
| 4          | Data Transfer Out  | Eliminar tráfego cross-region desnecessário e usar endpoints locais | 35%                  | 665                   | Médio   | Dependências ocultas entre aplicações           | Mapeamento de tráfego                 |
| 5          | NAT Gateway        | Consolidar gateways e criar VPC Endpoints para AWS Services         | 50%                  | 600                   | Médio   | Dependência de rotas atuais                     | Revisão de arquitetura de rede        |
| 6          | EKS                | Rightsizing de nodes e otimização do Cluster Autoscaler             | 8%                   | 536                   | Médio   | Escalabilidade insuficiente se mal configurado  | Métricas de utilização                |
| 7          | Lambda             | Ajustar memória e remover invocações desnecessárias                 | 40%                  | 360                   | Baixo   | Aumento do tempo de execução                    | Profiling das funções                 |
| 8          | S3 Standard        | Migrar objetos frios para Intelligent-Tiering ou Standard-IA        | 10%                  | 310                   | Baixo   | Aumento de custo de recuperação                 | Análise de acesso aos objetos         |
| 9          | RDS PostgreSQL     | Rightsizing e otimização de storage/IOPS                            | 3%                   | 246                   | Alto    | Impacto em performance do banco                 | Teste de carga e janela de manutenção |
| 10         | EBS gp3            | Remover volumes órfãos e ajustar provisionamento                    | 10%                  | 160                   | Baixo   | Exclusão indevida de volume                     | Inventário e backup                   |
| 11         | CloudWatch Metrics | Revisão de métricas customizadas pouco utilizadas                   | 15%                  | 135                   | Baixo   | Perda de visibilidade operacional               | Inventário de dashboards              |
| 12         | EC2 Reservada      | Ajustar futuras compras e cobertura de reservas                     | 5%                   | 210                   | Baixo   | Sem impacto imediato devido ao contrato vigente | Renovação futura                      |

### Resumo Financeiro

| Indicador          | Valor          |
| ------------------ | -------------- |
| Custo atual        | US$ 41.700/mês |
| Economia potencial | US$ 8.817/mês  |
| Redução potencial  | 21,1%          |
| Meta mínima        | 15%            |
| Resultado esperado | Meta superada  |

---

# Plano de execução até o final do trimestre (13 semanas)

## Semana 1

* Levantamento detalhado de custos via Cost Explorer.
* Habilitar tagging para identificação de workloads.
* Inventário de recursos EC2, EBS, Redis e RDS.
* Validar requisitos de retenção de logs com compliance.

**Entrega:** baseline de custos e utilização.

---

## Semana 2

* Identificar instâncias EC2 candidatas a rightsizing.
* Identificar volumes EBS órfãos.
* Levantar métricas de uso do Redis.
* Mapear tráfego entre regiões.

**Entrega:** backlog priorizado de otimizações.

---

## Semana 3

* Ajustar retenção do CloudWatch Logs.
* Limpar volumes EBS não utilizados.
* Revisar métricas customizadas do CloudWatch.

**Economia acumulada esperada:** ~4%.

---

## Semana 4

* Implementar Intelligent-Tiering nos buckets S3.
* Ajustar memória das funções Lambda.
* Revisar políticas de ciclo de vida do S3.

**Economia acumulada esperada:** ~5%.

---

## Semana 5

* Rightsizing do Redis em ambiente não produtivo.
* Testes de carga.
* Planejamento de rollout.

**Economia acumulada esperada:** ~7%.

---

## Semana 6

* Aplicar otimizações do Redis em produção.
* Revisar configuração do Cluster Autoscaler do EKS.

**Economia acumulada esperada:** ~9%.

---

## Semana 7

* Ajustar node groups do EKS.
* Remover capacidade ociosa persistente.

**Economia acumulada esperada:** ~10%.

---

## Semana 8

* Identificar workloads estáveis para Savings Plans.
* Simular cobertura ideal.

**Economia acumulada esperada:** ~10%.

---

## Semana 9

* Contratar Savings Plans.
* Migrar cargas elegíveis.

**Economia acumulada esperada:** ~15%.

---

## Semana 10

* Revisar arquitetura de NAT Gateway.
* Criar VPC Endpoints para S3, ECR, CloudWatch e STS.

**Economia acumulada esperada:** ~17%.

---

## Semana 11

* Consolidar NAT Gateways onde possível.
* Ajustar rotas.

**Economia acumulada esperada:** ~18%.

---

## Semana 12

* Eliminar tráfego cross-region desnecessário.
* Revisar replicações e integrações.

**Economia acumulada esperada:** ~20%.

---

## Semana 13

* Revisão final dos resultados.
* Atualização do forecast.
* Definição de governança contínua de custos.
* Criação de alertas de orçamento e FinOps.

**Resultado esperado ao final do trimestre**

* Economia recorrente: **US$ 8.000–8.800/mês**
* Redução total: **19%–21%**
* Sem degradação de SLA.
* Ambiente com monitoramento e governança de custos permanentes.

---

## Justificativa
```
A Task estebelece a análise dos dados do CSV com os valores do ambiente na AWS para mapeamento das oportunidades de economia, riscos e esforços; a Action especifica uma tabela técnica ordenada por impacto e um plano de execução semanal; a o Goal determina o objetivo final da tarefa que é a redução de 15% dos custos com Cloud até o final do trimestre sem comprometer o SLA e a performance.
```
