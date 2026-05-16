## Prompt:
```
# Before
Hoje temos o manifesto Kubernetes anexo Questão 05.yaml, referente à API Chronos, que foi escrito a 3 anos atrás.

# After
Queremos reescrever o manifesto yaml com as novas práticas da emprese:
Alta disponibilidade
Imagem versionada, não podendo usar latest
Não pode haver secrets no manifesto
Resources requests e limits
probes liveness e readness
Boas práticas de segurança (não-root)

# Bridge
Reescreva o manifesto com as novas práticas da empresa comentando cada etapa do manifesto

```
---

## Modelo:
```
GPT-5.5 e Opus 4.7
```
---

## Output GPT-5.5:
```
O manifesto atual tem vários problemas clássicos: 1 réplica (sem HA), uso de `latest`, secrets hardcoded, ausência de limites de recursos, nenhuma probe e falta de segurança básica.

Abaixo está uma versão reescrita seguindo boas práticas modernas de Kubernetes, com **comentários explicando cada parte**:

yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
  labels:
    app: chronos-api
spec:
  # Alta disponibilidade: mais de 1 réplica
  replicas: 3

  selector:
    matchLabels:
      app: chronos-api

  template:
    metadata:
      labels:
        app: chronos-api

    spec:
      # Boas práticas de segurança no nível do pod
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000

      containers:
      - name: api

        # Imagem versionada (nunca usar latest)
        image: chronos-api:v2.1.0

        ports:
        - containerPort: 8080

        # Remoção de secrets do manifesto
        # Agora referenciando Kubernetes Secrets
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: chronos-secrets
              key: db-password

        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: chronos-secrets
              key: jwt-secret

        # Definição de recursos (requests e limits)
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"

        # Liveness probe (verifica se o container precisa reiniciar)
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3

        # Readiness probe (verifica se está pronto para receber tráfego)
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3

        # Segurança no container
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
              - ALL

      # Evita concentração de pods no mesmo nó (melhora HA)
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: chronos-api
                topologyKey: "kubernetes.io/hostname"

---

## 🔍 Principais melhorias aplicadas

**Alta disponibilidade**

* Réplicas aumentadas para 3
* `podAntiAffinity` para distribuir pods entre nós

**Segurança**

* Execução como não-root
* Sistema de arquivos somente leitura
* Remoção de capabilities
* Sem secrets hardcoded

**Gestão de configuração**

* Uso de `Secret` (externo ao manifesto)
* Separação de responsabilidade (infra vs config sensível)

**Observabilidade e resiliência**

* `livenessProbe` → reinicia container travado
* `readinessProbe` → evita tráfego antes de estar pronto

**Governança de recursos**

* Requests e limits definidos (evita noisy neighbor e OOM)

**Boas práticas de imagem**

* Versão fixa (`v2.1.0`)
* Evita problemas de drift com `latest`

---

Se quiser, posso também te entregar:

* O YAML do `Secret` (seguindo boas práticas com base64 / sealed-secrets)
* Um `HorizontalPodAutoscaler` para escalar automaticamente
* Ou converter isso para Helm/Terraform para padronização enterprise

Só dizer 👍
```

## Output Opus 4.7:
```
# Reescrita do Manifesto Kubernetes — Chronos API

Vou analisar o manifesto original e reescrevê-lo aplicando as novas práticas. Antes de mostrar o resultado, vale destacar os principais problemas do manifesto atual:

**Problemas identificados:**
- `replicas: 1` → ponto único de falha
- `image: chronos-api:latest` → tag mutável, impossibilita rollback confiável
- `DB_PASSWORD` e `JWT_SECRET` em texto puro → exposição grave de credenciais
- Sem `resources` → risco de "noisy neighbor" e OOM imprevisível
- Sem probes → Kubernetes não sabe se o pod está saudável
- Sem `securityContext` → container roda como root por padrão

---

## Manifesto reescrito

yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
  labels:
    app: chronos-api
    version: v1.4.2
spec:
  # ─────────────────────────────────────────────────────────────
  # ALTA DISPONIBILIDADE
  # 3 réplicas permitem tolerância a falhas e rolling updates
  # sem downtime. Combinado com PodDisruptionBudget (recurso à
  # parte), garante que nunca caímos abaixo de N pods saudáveis.
  # ─────────────────────────────────────────────────────────────
  replicas: 3

  # Estratégia de atualização controlada: sobe 1 novo antes de
  # derrubar 1 antigo, mantendo capacidade durante o deploy.
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1

  selector:
    matchLabels:
      app: chronos-api

  template:
    metadata:
      labels:
        app: chronos-api
        version: v1.4.2
    spec:
      # ─────────────────────────────────────────────────────────
      # SEGURANÇA — securityContext no nível do POD
      # Define que TODO o pod roda sem privilégios de root.
      # runAsNonRoot: bloqueia o pod de subir se a imagem tentar
      #   rodar como UID 0.
      # fsGroup: garante que volumes montados pertençam ao grupo
      #   do usuário não-root da aplicação.
      # seccompProfile: ativa filtragem de syscalls (RuntimeDefault
      #   é o perfil seguro recomendado pela CNCF).
      # ─────────────────────────────────────────────────────────
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault

      # ─────────────────────────────────────────────────────────
      # ANTI-AFFINITY — Alta disponibilidade real
      # Espalha as réplicas por nós diferentes. Se um nó cair,
      # não perdemos todas as réplicas de uma vez.
      # ─────────────────────────────────────────────────────────
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: chronos-api
              topologyKey: kubernetes.io/hostname

      containers:
      - name: api

        # ───────────────────────────────────────────────────────
        # IMAGEM VERSIONADA
        # Tag imutável (v1.4.2). Nunca 'latest', pois:
        # - impede rollback determinístico
        # - cache do kubelet pode servir versão antiga
        # - dificulta auditoria do que está rodando em produção
        # ───────────────────────────────────────────────────────
        image: registry.empresa.com/chronos-api:v1.4.2
        imagePullPolicy: IfNotPresent

        ports:
        - name: http
          containerPort: 8080
          protocol: TCP

        # ───────────────────────────────────────────────────────
        # SEGURANÇA — securityContext no nível do CONTAINER
        # allowPrivilegeEscalation: false impede que o processo
        #   ganhe privilégios além dos do pai (bloqueia setuid).
        # readOnlyRootFilesystem: filesystem raiz em modo leitura;
        #   se a app precisa escrever (logs, cache), montamos
        #   volumes emptyDir específicos (ver abaixo).
        # capabilities: drop ALL remove todas as capabilities do
        #   Linux. A app HTTP não precisa de nenhuma.
        # ───────────────────────────────────────────────────────
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL

        # ───────────────────────────────────────────────────────
        # SECRETS — fora do manifesto
        # As credenciais agora vêm de um Secret do Kubernetes
        # (idealmente sincronizado de um cofre externo como
        # AWS Secrets Manager / HashiCorp Vault via
        # External Secrets Operator).
        # O manifesto referencia o Secret, mas nunca contém
        # o valor. Pode ser versionado em Git com segurança.
        # ───────────────────────────────────────────────────────
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: chronos-api-secrets
              key: db-password
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: chronos-api-secrets
              key: jwt-secret

        # ───────────────────────────────────────────────────────
        # RESOURCES — requests e limits
        # requests: garantia mínima (usada pelo scheduler para
        #   alocar o pod num nó com capacidade).
        # limits: teto absoluto. Acima disso, CPU é throttled e
        #   memória causa OOMKill.
        # Boa prática: memory request == memory limit (evita
        # eviction sob pressão). CPU limit pode ser maior que
        # request para permitir bursts.
        # ───────────────────────────────────────────────────────
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 512Mi

        # ───────────────────────────────────────────────────────
        # PROBES
        #
        # startupProbe: dá tempo para apps lentas iniciarem
        #   (JVM, carregamento de cache) sem que liveness mate
        #   o pod prematuramente. Enquanto roda, as outras
        #   probes ficam suspensas.
        #
        # readinessProbe: pod recebe tráfego? Se falhar, é
        #   removido do Service mas NÃO é reiniciado. Útil para
        #   pausas momentâneas (ex.: GC, dependência lenta).
        #
        # livenessProbe: pod está vivo? Se falhar, kubelet
        #   reinicia o container. Use endpoint que detecte
        #   deadlocks, não dependências externas (senão um DB
        #   fora derruba todos os pods em loop).
        # ───────────────────────────────────────────────────────
        startupProbe:
          httpGet:
            path: /health/startup
            port: http
          failureThreshold: 30
          periodSeconds: 10

        readinessProbe:
          httpGet:
            path: /health/ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3

        livenessProbe:
          httpGet:
            path: /health/live
            port: http
          initialDelaySeconds: 15
          periodSeconds: 20
          timeoutSeconds: 3
          failureThreshold: 3

        # ───────────────────────────────────────────────────────
        # Volumes graváveis exigidos por readOnlyRootFilesystem
        # Como o FS raiz é read-only, expomos diretórios
        # graváveis efêmeros via emptyDir apenas onde a app
        # realmente precisa escrever.
        # ───────────────────────────────────────────────────────
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /var/cache/chronos

      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
---

---

## Recursos complementares (recomendados)

O Deployment sozinho não cobre tudo. Para fechar o requisito de alta disponibilidade, vale subir ao lado:

**PodDisruptionBudget** — garante que durante manutenções voluntárias (drain de nó, upgrade de cluster) o número de pods saudáveis não caia abaixo do mínimo:

yaml
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: chronos-api-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: chronos-api
---

**Secret** (criado fora do Git, idealmente via External Secrets Operator puxando de um cofre):

yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: chronos-api-secrets
  namespace: production
type: Opaque
stringData:
  db-password: "<injetado pelo cofre>"
  jwt-secret: "<injetado pelo cofre>"
---

**HorizontalPodAutoscaler** — se a carga varia, escalar automaticamente complementa as 3 réplicas fixas.

---

## Pontos de atenção para implantar

Antes de aplicar em produção, alinhe com o time:

1. **Endpoints de health** — a aplicação precisa expor `/health/startup`, `/health/ready` e `/health/live`. Se hoje só existe um endpoint genérico, vale separar para que liveness não dependa do banco.
2. **UID 10001 no Dockerfile** — a imagem precisa criar esse usuário (`USER 10001`). Se a imagem atual roda como root, o pod não sobe.
3. **Resources** — os valores que coloquei (250m CPU / 512Mi RAM) são um ponto de partida. O ideal é medir o consumo real com `kubectl top` ou Prometheus por alguns dias e ajustar.
4. **Secret backend** — definir como o Secret será populado: `kubectl create secret` é aceitável para começar, mas o objetivo de médio prazo é integração com Vault/AWS Secrets Manager via ESO.
```
---

## Justificativa:
```
O Before traz o contexto de cenário atual do manifesto Kubernetes com diversos problemas; o After define a reescrita do manifesto Kubernetes corrigindo os problemas e aplicando as melhores práticas de escrita de manifestos; o Bridge tem a ação pedida de reescrever o manifesto conforme as orientações. O prompt traz instruções claras das correções desejadas.
```
