## Prompt:
```
# Role
Você é um SRE sênior com larga experiência em AWS, Kubernetes, ArgoCD

# Input
Chronos roda no EKS, namespace production, 6 réplicas com HPA configurado (min 4, max 12, CPU target 70%).
Deploy via Argo CD a partir do repositório hvt/chronos-api.
Dependências diretas: Ledger (PostgreSQL) e Reactor (filas SQS).
Observabilidade: métricas expostas em /metrics, logs centralizados no Beacon, dashboards em Grafana.
Ferramentas disponíveis para o plantão: kubectl, aws cli, argocd cli.
Canal de plantão: #oncall-chronos no Slack.
Time sênior de escalação: @chronos-core (SLA de resposta: 15 minutos em horário comercial, 30 fora). 

# Steps
Confirmar o estado atual dos pods
Analisar as métricas de memória no /metrics e correlacionar com os logs no Beacon
Verificar se o problema é no pod da API ou no ingress
Avaliar se o alerta é devido à utilização da API ou se está subdimensionado
Descrever ações imediatas de resolução
Definir critérios de acionamento e excalonamento
Listar critérios de validação pós correção
Proposta de correção definitiva do problema

# Expectation
Escreva um runbook completo com:
Comandos kubectl prontos para copiar e colar para cada etapa com um exemplo da saída esperada para cada comando
Queries para análise do consumo de recursos
Árvore de decisão: leak de memória x subdimencionamento com ações bem definidas a serem executadas em cada situação
Tempo limite em que cada etapa pode aguardar com as intruções claras da ação em caso de o tempo estipulado se encerrar
Critérios bem definidos para decisão de quando o chamado precisa ser escalado
Definição dos critérios de rollback bem definidos

```
---

## Modelo:
```
Gemini 3.1 Pro e GPT-5.5
```
---

## Output Gemini 3.1 Pro:

## Runbook de Incidente: API Chronos (Memória/Recursos)

Este runbook define os procedimentos padrão para diagnóstico e mitigação de incidentes relacionados a consumo de recursos e indisponibilidade no serviço **Chronos** (namespace `production`).

---

### 1. Confirmação do Estado Atual dos Pods

**⏳ Tempo Limite:** 5 Minutos
**⚠️ Ação caso o tempo expire:** Se a API estiver 100% indisponível e você não conseguir confirmar o estado, execute o Rollback para a última versão estável (Ver Seção 6) e escale o incidente.

Inicie verificando a saúde dos pods e o comportamento do HPA.

```bash
# Verificar status dos pods (buscando por CrashLoopBackOff ou OOMKilled)
kubectl get pods -n production -l app=chronos

```

**Saída Esperada:**

```text
NAME                              READY   STATUS             RESTARTS      AGE
chronos-api-6b7df9f9c-2x8jk       1/1     Running            0             2d
chronos-api-6b7df9f9c-5k9lm       0/1     OOMKilled          2 (30s ago)   5m
chronos-api-6b7df9f9c-8p1qr       1/1     Running            0             2d
...

```

```bash
# Verificar o status do HPA e consumo de CPU
kubectl get hpa -n production chronos-api

```

**Saída Esperada:**

```text
NAME          REFERENCE                TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
chronos-api   Deployment/chronos-api   45%/70%   4         12        6          30d

```

*(Nota: O HPA está configurado para CPU. Se os pods estiverem sofrendo OOMKilled sem pico de CPU, o HPA não escalará a aplicação automaticamente).*

---

### 2. Isolamento da Falha: Pod da API ou Ingress?

**⏳ Tempo Limite:** 5 Minutos
**⚠️ Ação caso o tempo expire:** Escale imediatamente no canal `#oncall-chronos`.

Determine se o gargalo está na recepção do tráfego (Ingress) ou no processamento (Pod).

```bash
# Verificar os endpoints ativos (se os pods estão recebendo tráfego)
kubectl get endpoints chronos-api -n production

```

**Saída Esperada:**

```text
NAME          ENDPOINTS                                         AGE
chronos-api   10.0.1.15:8080,10.0.2.33:8080,10.0.3.12:8080...   30d

```

```bash
# Analisar logs recentes do Ingress Controller (ex: NGINX/ALB) filtrando por chronos
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=500 | grep "chronos-api"

```

* **Se Ingress reporta HTTP 502/504 e os Pods estão reiniciando:** O problema está nos Pods (provável OOMKilled ou travamento por falta de recursos).
* **Se Ingress reporta erros, mas Pods estão saudáveis e com baixo consumo:** O problema pode estar no Ingress Controller (esgotamento de conexões) ou na rede.

---

### 3. Análise de Métricas e Logs

**⏳ Tempo Limite:** 10 Minutos
**⚠️ Ação caso o tempo expire:** Se a causa não for identificada, aplique a mitigação imediata de reiniciar o deployment (Step 4) para restabelecer o serviço enquanto analisa.

#### Queries para Análise de Recursos (Prometheus / Grafana)

**1. Consumo de Memória por Pod (Working Set):**

```promql
sum(container_memory_working_set_bytes{namespace="production", pod=~"chronos-api-.*", container="chronos-api"}) by (pod)

```

**2. OOMKilled Rate:**

```promql
rate(kube_pod_container_status_restarts_total{namespace="production", pod=~"chronos-api-.*"}[5m]) > 0

```

**3. Utilização do HPA (CPU):**

```promql
avg(node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate{namespace="production", pod=~"chronos-api-.*"}) / avg(kube_pod_container_resource_requests{namespace="production", pod=~"chronos-api-.*", resource="cpu"})

```

#### Correlação com Logs (Beacon / Elasticsearch / Loki)

Busque no Beacon as seguintes queries (sintaxe adaptada para o seu stack de log):

* `app: "chronos-api" AND level: "ERROR" AND message: "*memory*"`
* `app: "chronos-api" AND message: "*timeout*"` (Pode indicar contenção no PostgreSQL/Ledger ou SQS/Reactor gerando acúmulo de requisições em memória).

---

### 4. Árvore de Decisão: Memory Leak vs Subdimensionamento

Com base nas métricas acima, utilize a tabela abaixo para determinar o cenário e a ação imediata.

| Sintoma Observado (Métricas/Logs) | Diagnóstico | Ação Imediata |
| --- | --- | --- |
| Memória cresce em "escadinha" (linear) ao longo dos dias, independente do tráfego. CPU estável. OOMKilled ocorre periodicamente. | **Memory Leak** (Vazamento de Memória) | Reiniciar pods para alívio imediato.<br>

<br>`kubectl rollout restart deploy chronos-api -n production` |
| Memória dá "spikes" súbitos que correlacionam exatamente com picos de requisições no Ingress ou mensagens presas no SQS (Reactor). | **Subdimensionamento** (Tráfego > Capacidade) | Desabilitar sync do ArgoCD temporariamente e aumentar limite de memória/replicas.<br>

<br>`argocd app set chronos-api --sync-policy none`<br>

<br>`kubectl scale deploy chronos-api --replicas=12 -n production` |
| Memória alta associada a logs de timeout de banco (Ledger). Conexões travadas aguardando I/O. | **Gargalo de Dependência** (Ledger) | Avaliar saúde do PostgreSQL. Aumentar timeout do client na API Chronos para dropar requisições mais rápido (Evita acúmulo em memória). |

---

### 5. Critérios de Ação, Rollback e Escalonamento

#### Critérios de Rollback

O rollback deve ser executado **imediatamente** se:

1. Houve um deploy nas últimas 2 horas (via Argo CD) e a taxa de erros HTTP 5xx ultrapassou 5%.
2. A aplicação entrou em `CrashLoopBackOff` generalizado e não estabiliza após 5 minutos.

**Comando de Rollback (Argo CD):**

```bash
# Listar o histórico para pegar o ID da revisão anterior
argocd app history chronos-api

# Executar o rollback
argocd app rollback chronos-api <REVISION_ID>

```

#### Critérios de Escalonamento (Acionar `@chronos-core`)

Acione o time sênior no `#oncall-chronos` (SLA 15m comercial / 30m plantão) se **qualquer** das condições abaixo for verdadeira:

1. O problema persistir após o Rollback.
2. A indisponibilidade for causada por queda nas dependências críticas (Ledger PostgreSQL indisponível ou SQS inacessível na AWS).
3. Esgotamento do tempo limite de investigação (Timeboxes estourados) sem identificação clara da causa.
*Mensagem padrão para o Slack:* `@chronos-core Incidente crítico na API Chronos. Rollback/Restart executado sem sucesso. Sintoma: [OOMKilled / Timeout Ingress]. Grafana: [Link]. Logs: [Link].`

---

### 6. Validação Pós-Correção e Proposta Definitiva

#### Critérios de Validação Pós-Correção

Após aplicar a mitigação, valide a estabilidade do sistema nos próximos 15 minutos:

1. **Pods:** Todos os pods devem estar `1/1 Running` há pelo menos 10 minutos sem novos restarts.
2. **Taxa de Erro:** Retorno de HTTP 5xx no Ingress deve ser menor que 0.1%.
3. **Métricas:** Consumo de CPU abaixo de 70% (target do HPA) e memória estabilizada abaixo de 80% do `limit` estipulado.
4. **Dependências:** Conexões com o PostgreSQL (Ledger) e SQS (Reactor) estabelecidas e processando a fila sem lag acumulado.

#### Proposta de Correção Definitiva (Post-Mortem Action Items)

Dependendo da causa raiz confirmada, a engenharia deve priorizar as seguintes ações no repositório `hvt/chronos-api`:

* **Se Memory Leak:**
* Gerar um heap dump da aplicação durante o pico de memória.
* Revisar código recente focado em escopo de variáveis, fechamento de conexões de banco de dados (`defer rows.Close()`, caso Go, ou garbage collection tuning dependendo da linguagem).


* **Se Subdimensionamento / Picos Súbitos:**
* **HPA Tuning:** Alterar o HPA para escalar também baseado em consumo de memória, não apenas CPU (ex: `type: Resource, name: memory, targetAverageUtilization: 75`).
* **KEDA:** Avaliar a substituição do HPA padrão pelo KEDA, configurando triggers baseados no tamanho da fila SQS (Reactor), permitindo escalonamento proativo antes do gargalo de memória ocorrer.
* **Resource Limits:** Ajustar os manifests no ArgoCD com valores realistas de `requests` e `limits` validados por testes de carga (Load Testing).

---
## Output GPT-5.5:

Abaixo está um **runbook completo, operacional e direto para uso em plantão** para o cenário descrito.

---

# 📘 RUNBOOK — Incidente de Memória / Saturação na Chronos API (EKS)

## 🎯 Objetivo

Identificar se o alerta é causado por **leak de memória, subdimensionamento ou problema de infra (ingress/dependências)** e aplicar ações imediatas com critérios claros de escalonamento e rollback.

---

# ⏱️ SLA DE EXECUÇÃO (tempo máximo por etapa)

| Etapa                        | Tempo limite |
| ---------------------------- | ------------ |
| Verificar pods               | 2 min        |
| Métricas + logs              | 5 min        |
| Isolar API vs Ingress        | 3 min        |
| Diagnóstico (árvore decisão) | 5 min        |
| Ação corretiva               | 5 min        |
| Validação                    | 10 min       |

👉 **Se ultrapassar 15 min sem diagnóstico claro → ESCALAR**

---

# 1️⃣ Verificar estado dos pods

```bash
kubectl get pods -n production -l app=chronos-api
```

### ✅ Saída esperada

```
NAME                          READY   STATUS    RESTARTS   AGE
chronos-api-abc123            1/1     Running   0          2h
chronos-api-def456            1/1     Running   1          2h
```

### 🚨 Sinais de problema

* `CrashLoopBackOff`
* `OOMKilled`
* `Restart > 3`

```bash
kubectl describe pod <pod> -n production
```

### Procurar:

```
Reason: OOMKilled
```

---

# 2️⃣ Analisar consumo de memória

```bash
kubectl top pods -n production -l app=chronos-api
```

### ✅ Exemplo

```
NAME                     CPU(m)   MEMORY(Mi)
chronos-api-abc123       200m     850Mi
chronos-api-def456       210m     900Mi
```

---

## 🔍 Métricas detalhadas (/metrics)

```bash
kubectl port-forward svc/chronos-api 8080:80 -n production
curl localhost:8080/metrics | grep -i memory
```

### Queries úteis (Prometheus / Grafana)

```
container_memory_working_set_bytes{pod=~"chronos-api.*"}
rate(container_memory_usage_bytes[5m])
```

👉 Indicadores:

* Crescimento contínuo → possível leak
* Pico + estabilização → carga

---

# 3️⃣ Correlacionar com logs (Beacon)

### Query exemplo:

```
service:chronos-api AND ("OOM" OR "memory" OR "killed")
```

👉 Verificar:

* OutOfMemory
* Timeout downstream (Ledger/Reactor)
* Backpressure

---

# 4️⃣ Isolar problema: API vs Ingress

```bash
kubectl get ingress -n production
kubectl describe ingress <ingress-name>
```

```bash
kubectl logs -n production deploy/chronos-api --tail=100
```

### Teste direto no pod (bypass ingress)

```bash
kubectl port-forward pod/<pod> 8081:8080 -n production
curl localhost:8081/health
```

👉 Resultado:

* Funciona direto → problema no ingress
* Falha → problema na API

---

# 5️⃣ Árvore de decisão

## 🌳 Diagnóstico

### 📌 Caso A — Leak de memória

**Sinais:**

* Memória cresce continuamente
* OOMKilled frequente
* Não reduz após GC

👉 Ação:

```bash
kubectl rollout restart deploy chronos-api -n production
```

👉 Mitigação:

* Reduzir impacto com scaling:

```bash
kubectl scale deploy chronos-api --replicas=10 -n production
```

---

### 📌 Caso B — Subdimensionamento

**Sinais:**

* Memória alta porém estável
* HPA próximo do máximo
* Sem OOM

👉 Ação:

```bash
kubectl edit hpa chronos-api -n production
```

Alterar:

```
maxReplicas: 12 → 16
```

OU ajuste rápido:

```bash
kubectl scale deploy chronos-api --replicas=12 -n production
```

---

### 📌 Caso C — Problema externo (Ledger / SQS)

**Sinais:**

* Logs com timeout
* Latência alta
* Fila acumulando

👉 Ação:

```bash
aws sqs get-queue-attributes --queue-url <url> --attribute-names ApproximateNumberOfMessages
```

👉 Mitigação:

* Escalar workers
* Reduzir pressão na API

---

# 6️⃣ Ações imediatas

| Situação          | Ação                       |
| ----------------- | -------------------------- |
| OOMKilled         | Restart pods               |
| Alta carga        | Scale up                   |
| Ingress falha     | Restart ingress controller |
| Dependência lenta | Reduzir throughput / retry |

---

# 7️⃣ Critérios de escalonamento

🚨 Escalar para **@chronos-core** se:

* OOM persiste após restart
* HPA no máximo por >10 min
* Erros >5% por 5 min
* Latência > p95 esperado por 10 min
* Dependência crítica fora

⏱️ SLA:

* 15 min (horário comercial)
* 30 min (fora)

---

# 8️⃣ Critérios de rollback

Executar rollback via ArgoCD:

```bash
argocd app history chronos-api
argocd app rollback chronos-api <revision>
```

👉 Fazer rollback se:

* Incidente começou após deploy recente
* Erros aumentaram após nova versão
* Consumo de memória anormal comparado baseline

---

# 9️⃣ Validação pós-correção

### Checklist

* Pods estáveis (sem restart)

```bash
kubectl get pods -n production
```

* Memória estabilizada

```
container_memory_usage_bytes não crescente
```

* Latência normal (Grafana)

* Erros < 1%

* Fila SQS normalizada

---

# 🔧 Correção definitiva (pós-mortem)

## Se leak de memória:

* Heap profiling
* Ajustar GC / runtime
* Adicionar limites:

```yaml
resources:
  limits:
    memory: "1Gi"
```

---

## Se subdimensionamento:

* Ajustar HPA com base em memória também

```yaml
metrics:
- type: Resource
  resource:
    name: memory
```

---

## Se arquitetura:

* Circuit breaker
* Retry com backoff
* Cache

---

# 📣 Comunicação (Slack)

Canal: `#oncall-chronos`

Mensagem padrão:

```
[INCIDENTE] Chronos API com alto consumo de memória
Status: Mitigando
Ação: Scaling / Restart
Impacto: Latência elevada
Próxima atualização: 10 min
```

---

# ✅ Resumo operacional

1. Ver pods
2. Ver memória
3. Ver logs
4. Isolar API vs ingress
5. Classificar (leak vs carga)
6. Mitigar rápido
7. Validar
8. Escalar se necessário

---

Se quiser, posso transformar isso em **playbook automatizado (script bash + queries prontas + integração com Slack/ArgoCD)**.

---

## Justificativa:
```
A Role define claramente o nível de conhecimento requerido para análise do prompt; o Input define um contexto detalhado de como é o ambiente analisado; o Steps define quais os pontos que são necessários serem abordados para elaboração da documentação; o Expectation define o que é esperado na saída do prompt, no caso um runbook com os campos necessários.
```
