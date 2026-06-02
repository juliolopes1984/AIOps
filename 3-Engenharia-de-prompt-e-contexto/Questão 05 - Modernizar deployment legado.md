## Prompt:
```
# Before
Hoje temos o manifesto Kubernetes abaixo, referente à API Chronos, que foi escrito a 3 anos atrás.

apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: chronos-api
  template:
    metadata:
      labels:
        app: chronos-api
    spec:
      containers:
      - name: api
        image: chronos-api:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_PASSWORD
          value: "P@ssw0rd2023!"
        - name: JWT_SECRET
          value: "hvt-jwt-prod-secret"


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
GPT-5.5 e Opus 4.8
```
---

## Output GPT-5.5:

Abaixo está uma versão modernizada do manifesto seguindo as práticas solicitadas: alta disponibilidade, imagem versionada, remoção de secrets do YAML, recursos definidos, probes e execução sem privilégios.

```yaml
# Deployment da API Chronos
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
  labels:
    app: chronos-api

spec:
  # Alta disponibilidade com múltiplas réplicas
  replicas: 3

  # Estratégia de atualização sem indisponibilidade
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

    spec:
      # Segurança em nível de Pod
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001

      containers:
        - name: api

          # Imagem versionada (nunca usar latest)
          image: chronos-api:v2.4.1

          imagePullPolicy: IfNotPresent

          ports:
            - containerPort: 8080
              name: http

          # Variáveis não sensíveis
          env:
            - name: APP_ENV
              value: "production"

          # Secrets obtidos do Kubernetes Secret
          envFrom:
            - secretRef:
                name: chronos-api-secrets

          # Garantia mínima de recursos
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"

            limits:
              cpu: "1000m"
              memory: "1Gi"

          # Verifica se a aplicação está pronta para receber tráfego
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3

          # Verifica se a aplicação continua saudável
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            initialDelaySeconds: 30
            periodSeconds: 15
            timeoutSeconds: 3
            failureThreshold: 3

          # Hardening do container
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL

---
# Distribui os pods entre nós para aumentar disponibilidade
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: chronos-api
  namespace: production

spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: chronos-api

---
# Secret apenas como exemplo de referência.
# Recomenda-se gerenciamento via External Secrets,
# AWS Secrets Manager ou Vault.
apiVersion: v1
kind: Secret
metadata:
  name: chronos-api-secrets
  namespace: production
type: Opaque
stringData:
  DB_PASSWORD: "<managed-externally>"
  JWT_SECRET: "<managed-externally>"
```

### Comentários das alterações

| Requisito                | Implementação                                                                  | Benefício                                             |
| ------------------------ | ------------------------------------------------------------------------------ | ----------------------------------------------------- |
| Alta disponibilidade     | `replicas: 3` + `PodDisruptionBudget`                                          | Mantém a API disponível durante falhas ou manutenções |
| Imagem versionada        | `chronos-api:v2.4.1`                                                           | Garante rastreabilidade e rollback                    |
| Sem secrets no manifesto | `envFrom.secretRef`                                                            | Evita exposição de credenciais no Git                 |
| Requests e Limits        | CPU e memória definidos                                                        | Evita contenção de recursos e melhora o agendamento   |
| Readiness Probe          | `/health/ready`                                                                | Tráfego enviado apenas para pods prontos              |
| Liveness Probe           | `/health/live`                                                                 | Reinício automático em caso de travamento             |
| Não-root                 | `runAsNonRoot: true`                                                           | Reduz superfície de ataque                            |
| Hardening                | `allowPrivilegeEscalation: false`, `drop: ALL`, `readOnlyRootFilesystem: true` | Melhora a segurança do container                      |
| Rolling Update           | `maxUnavailable: 0`                                                            | Atualizações sem downtime                             |

Para um ambiente EKS de produção, eu também recomendaria complementar com **HPA (Horizontal Pod Autoscaler)**, **Topology Spread Constraints**, **NetworkPolicy** e integração com **AWS Secrets Manager + External Secrets Operator**, elevando o manifesto para um padrão próximo ao utilizado em ambientes enterprise.


---

## Output Opus 4.8:

Boa! Vou reescrever o manifesto aplicando todas as práticas que você listou, com comentários explicando cada decisão. Vou entregar como arquivo `.yaml` pronto para uso.Pronto. Resumo do que mudou em relação ao manifesto antigo:

**Alta disponibilidade** — subi de 1 para 3 réplicas, adicionei `podAntiAffinity` (para espalhar os pods entre nós diferentes), uma `strategy` de rollout sem downtime (`maxUnavailable: 0`) e um `PodDisruptionBudget` que protege o serviço durante manutenções de nós.

**Imagem versionada** — troquei `chronos-api:latest` por `chronos-api:1.4.2`. Em produção, considerar usar o digest (`@sha256:...`) para imutabilidade total.

**Sem secrets no manifesto** — `DB_PASSWORD` e `JWT_SECRET` agora vêm de `secretKeyRef`, apontando para objetos `Secret` criados à parte. O manifesto só referencia, nunca expõe o valor.

**Resources** — `requests` (o que o scheduler reserva) e `limits` (o teto). Os números são um ponto de partida; ajuste com base no consumo real medido.

**Probes** — `liveness` (reinicia se travar) e `readiness` (tira do balanceamento sem reiniciar), em endpoints distintos.

**Não-root / segurança** — `securityContext` no pod (`runAsNonRoot`, UID/GID explícitos, `seccompProfile`) e no container (`allowPrivilegeEscalation: false`, `readOnlyRootFilesystem`, `drop: ALL`). Como o filesystem fica somente leitura, adicionei um volume `emptyDir` em `/tmp` para o que a aplicação precisar escrever.

Dois pontos que dependem da sua aplicação para finalizar: confirme os **caminhos das probes** (`/healthz` e `/ready`) e os **nomes/chaves dos Secrets** (`chronos-db-credentials`/`password` e `chronos-jwt`/`secret`). E lembre que os objetos `Secret` precisam existir no cluster — quer que eu monte um exemplo de como criá-los via Sealed Secrets ou External Secrets?

```
# ============================================================================
# Chronos API - Deployment (reescrito com as novas práticas da empresa)
# ============================================================================

apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
  labels:
    app: chronos-api
    # Labels recomendadas pela documentação do Kubernetes (app.kubernetes.io/*)
    # ajudam em observabilidade, seleção e ferramentas de GitOps/Helm.
    app.kubernetes.io/name: chronos-api
    app.kubernetes.io/part-of: chronos
    app.kubernetes.io/managed-by: kubectl
spec:
  # --------------------------------------------------------------------------
  # ALTA DISPONIBILIDADE
  # Antes era 1 réplica (ponto único de falha). Com 3 réplicas, o serviço
  # continua de pé se um pod/nó cair. Idealmente este valor é controlado por
  # um HorizontalPodAutoscaler (HPA) em vez de fixo.
  # --------------------------------------------------------------------------
  replicas: 3

  # Atualização sem downtime: sobe novos pods antes de derrubar os antigos.
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0   # nunca fica com menos pods saudáveis que o desejado
      maxSurge: 1         # cria no máximo 1 pod extra durante o rollout

  selector:
    matchLabels:
      app: chronos-api

  template:
    metadata:
      labels:
        app: chronos-api
    spec:
      # ----------------------------------------------------------------------
      # SEGURANÇA NO NÍVEL DO POD (não-root)
      # Define que TODOS os containers do pod rodam como usuário sem privilégio.
      # ----------------------------------------------------------------------
      securityContext:
        runAsNonRoot: true        # bloqueia o pod se a imagem tentar rodar como root
        runAsUser: 10001          # UID não-root explícito
        runAsGroup: 10001
        fsGroup: 10001            # grupo dono dos volumes montados
        seccompProfile:
          type: RuntimeDefault    # aplica o perfil seccomp padrão do runtime

      # ----------------------------------------------------------------------
      # ALTA DISPONIBILIDADE - espalhar réplicas entre nós diferentes
      # Anti-afinidade evita que todas as réplicas caiam junto se um nó falhar.
      # ----------------------------------------------------------------------
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
          # ------------------------------------------------------------------
          # IMAGEM VERSIONADA (sem :latest)
          # 'latest' é imprevisível: dois pods podem subir versões diferentes e
          # o rollback fica impossível. Use uma tag imutável (semver ou digest).
          # ------------------------------------------------------------------
          image: chronos-api:1.4.2
          imagePullPolicy: IfNotPresent

          ports:
            - name: http
              containerPort: 8080

          # ------------------------------------------------------------------
          # SECRETS FORA DO MANIFESTO
          # Nada de senha/segredo em texto puro no YAML. Os valores vêm de
          # objetos Secret, criados/gerenciados à parte (Vault, Sealed Secrets,
          # External Secrets, etc.). O manifesto só REFERENCIA o Secret.
          # ------------------------------------------------------------------
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: chronos-db-credentials
                  key: password
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: chronos-jwt
                  key: secret

          # ------------------------------------------------------------------
          # RESOURCES (requests e limits)
          # requests = o que o scheduler reserva (garante onde cabe o pod).
          # limits   = teto que o container não ultrapassa (evita "vizinho
          #            barulhento" e OOM derrubando o nó inteiro).
          # Ajuste os valores conforme o perfil real de consumo da aplicação.
          # ------------------------------------------------------------------
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi

          # ------------------------------------------------------------------
          # PROBES
          # liveness  -> "o processo travou?" Se falhar, o Kubernetes reinicia.
          # readiness -> "está pronto pra receber tráfego?" Se falhar, o pod é
          #              tirado do balanceamento sem ser reiniciado.
          # Use endpoints distintos para cada uma.
          # ------------------------------------------------------------------
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 15
            periodSeconds: 20
            timeoutSeconds: 3
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3

          # ------------------------------------------------------------------
          # SEGURANÇA NO NÍVEL DO CONTAINER
          # Endurece o container individualmente.
          # ------------------------------------------------------------------
          securityContext:
            allowPrivilegeEscalation: false  # impede ganho de privilégio (ex: setuid)
            readOnlyRootFilesystem: true     # filesystem raiz somente leitura
            capabilities:
              drop:
                - ALL                        # remove todas as capabilities do Linux

          # readOnlyRootFilesystem exige volumes graváveis para diretórios
          # temporários que a app realmente precisa escrever.
          volumeMounts:
            - name: tmp
              mountPath: /tmp

      volumes:
        - name: tmp
          emptyDir: {}

---
# ============================================================================
# PodDisruptionBudget - complemento de ALTA DISPONIBILIDADE
# Garante que, em manutenções/drenagem de nós, pelo menos 2 pods continuem
# disponíveis (o cluster não pode derrubar todos de uma vez).
# ============================================================================
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: chronos-api
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: chronos-api
```

---

## Justificativa:
```
O Before traz o contexto de cenário atual do manifesto Kubernetes com diversos problemas; o After define a reescrita do manifesto Kubernetes corrigindo os problemas e aplicando as melhores práticas de escrita de manifestos; o Bridge tem a ação pedida de reescrever o manifesto conforme as orientações. O prompt traz instruções claras das correções desejadas.
```
