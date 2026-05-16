## Prompt
```
# Role
Você é um SRE com vasto conhecimento e experiência em Linux, Bash, PostgreSQL e AWS

# Task
Crie um script bash de backup do banco PostgreSQL de nome ledger_prod que roda em uma EC2 Ubuntu 22.04 criada na região us-east-1, utilizando o diretório de trabalho /var/backups/ledger com 80GB livres, URL do host é ledger-db.internal.hvt.io, porta de serviço do banco 5432, utilize o usuário backup_user para autenticação no PostgreSQL, a senha está configurada na variável de ambiente PGPASSWORD que é populada pelo AWS Secrets Manager via IAM role da instância, o backup deve ser realizado utilizando a ferramenta pg_dump e o backup compactado utilizando gzip, após a compactação, subir o backup para o bucket S3 hvt-ledger-backups utilizando o aws s3 cp, retenção de 30 dias no S3, os arquivos com mais de 30 dias no S3 devem ser removidos, cada execussão deve ser registrada no arquivo /var/log/ledger-backup.log com timestamp e exit code em caso de falha. Crie uma instrução de agendamento diário do backup às 22h utilizando ferramenta nativa do sistema operacional

# Format
Retorne o script bash e as orientações de agendamento do backup, não retorne explicações
```
---

## Modelo
```
Gemini 3 Flash e Sonet 4.6
```
---

## Output Gemini 3 Flash
```
#!/bin/bash

# Configurações
DB_NAME="ledger_prod"
DB_HOST="ledger-db.internal.hvt.io"
DB_PORT="5432"
DB_USER="backup_user"
BACKUP_DIR="/var/backups/ledger"
S3_BUCKET="s3://hvt-ledger-backups"
LOG_FILE="/var/log/ledger-backup.log"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30
FILE_NAME="${DB_NAME}_${TIMESTAMP}.sql.gz"
FILE_PATH="${BACKUP_DIR}/${FILE_NAME}"

# Garantir diretório de trabalho e permissões de log
mkdir -p "$BACKUP_DIR"
touch "$LOG_FILE"

exec_log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOG_FILE"
}

# Execução do Backup
pg_dump -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" "$DB_NAME" | gzip > "$FILE_PATH"
EXIT_CODE=$?

if [ $EXIT_CODE -ne 0 ]; then
    exec_log "FAILURE: pg_dump falhou com exit code ${EXIT_CODE}"
    exit $EXIT_CODE
fi

# Upload para S3
aws s3 cp "$FILE_PATH" "${S3_BUCKET}/${FILE_NAME}" --region us-east-1 >> "$LOG_FILE" 2>&1
UPLOAD_EXIT=$?

if [ $UPLOAD_EXIT -eq 0 ]; then
    exec_log "SUCCESS: Backup enviado para o S3: ${FILE_NAME}"
    rm -f "$FILE_PATH"
else
    exec_log "FAILURE: Upload para S3 falhou com exit code ${UPLOAD_EXIT}"
    exit $UPLOAD_EXIT
fi

# Remoção de arquivos antigos no S3 (Retenção de 30 dias)
PURGE_DATE=$(date -d "${RETENTION_DAYS} days ago" +%Y-%m-%d)

aws s3api list-objects-v2 --bucket "${S3_BUCKET#s3://}" --query "Contents[?LastModified<='${PURGE_DATE}'].{Key: Key}" --output text | xargs -I {} aws s3 rm "${S3_BUCKET}/{}" >> "$LOG_FILE" 2>&1

exec_log "COMPLETED: Processo de backup finalizado."
```
```
Agendamento (Crontab)
Para configurar a execução diária às 22h, edite o crontab do usuário com permissão de escrita nos diretórios (ex: root ou usuário com permissão no /var/backups/ledger):

Abra o editor:

Bash
sudo crontab -e
Adicione a seguinte linha ao final do arquivo (ajuste o caminho do script conforme necessário):

Snippet de código
00 22 * * * /bin/bash /usr/local/bin/ledger-backup.sh
Garanta que o script tenha permissão de execução:

Bash
sudo chmod +x /usr/local/bin/ledger-backup.sh
```

## Output Sonet 4.6
```
#!/usr/bin/env bash
# ==============================================================================
# ledger_backup.sh — PostgreSQL backup for ledger_prod → S3
# ==============================================================================
set -euo pipefail

# ------------------------------------------------------------------------------
# Configuration
# ------------------------------------------------------------------------------
DB_HOST="ledger-db.internal.hvt.io"
DB_PORT="5432"
DB_NAME="ledger_prod"
DB_USER="backup_user"

WORK_DIR="/var/backups/ledger"
LOG_FILE="/var/log/ledger-backup.log"
S3_BUCKET="s3://hvt-ledger-backups"
RETENTION_DAYS=30
MIN_FREE_GB=10

TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_FILE="${WORK_DIR}/${DB_NAME}_${TIMESTAMP}.sql.gz"
S3_KEY="${DB_NAME}/${DB_NAME}_${TIMESTAMP}.sql.gz"

# ------------------------------------------------------------------------------
# Logging helpers
# ------------------------------------------------------------------------------
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "${LOG_FILE}"
}

log_exit() {
    local exit_code=$1
    local message=$2
    log "EXIT_CODE=${exit_code} | ${message}"
    exit "${exit_code}"
}

# ------------------------------------------------------------------------------
# Preflight checks
# ------------------------------------------------------------------------------
preflight_checks() {
    log "INFO  | Starting preflight checks"

    for cmd in pg_dump gzip aws; do
        if ! command -v "${cmd}" &>/dev/null; then
            log_exit 1 "FAIL  | Required command not found: ${cmd}"
        fi
    done

    mkdir -p "${WORK_DIR}"

    local free_gb
    free_gb=$(df -BG "${WORK_DIR}" | awk 'NR==2 {gsub("G",""); print $4}')
    if (( free_gb < MIN_FREE_GB )); then
        log_exit 1 "FAIL  | Insufficient disk space: ${free_gb}GB free (minimum ${MIN_FREE_GB}GB required)"
    fi

    if [[ -z "${PGPASSWORD:-}" ]]; then
        log_exit 1 "FAIL  | PGPASSWORD environment variable is not set"
    fi

    log "INFO  | Preflight checks passed (free disk: ${free_gb}GB)"
}

# ------------------------------------------------------------------------------
# Dump + compress
# ------------------------------------------------------------------------------
run_backup() {
    log "INFO  | Dumping database ${DB_NAME} from ${DB_HOST}:${DB_PORT}"

    if pg_dump \
        --host="${DB_HOST}" \
        --port="${DB_PORT}" \
        --username="${DB_USER}" \
        --no-password \
        --format=plain \
        --blobs \
        "${DB_NAME}" \
        | gzip -9 > "${BACKUP_FILE}"; then

        local size
        size=$(du -sh "${BACKUP_FILE}" | cut -f1)
        log "INFO  | Backup created: ${BACKUP_FILE} (${size})"
    else
        rm -f "${BACKUP_FILE}"
        log_exit 2 "FAIL  | pg_dump/gzip pipeline failed"
    fi
}

# ------------------------------------------------------------------------------
# Upload to S3
# ------------------------------------------------------------------------------
upload_to_s3() {
    log "INFO  | Uploading to ${S3_BUCKET}/${S3_KEY}"

    if aws s3 cp "${BACKUP_FILE}" "${S3_BUCKET}/${S3_KEY}" \
        --storage-class STANDARD_IA \
        --no-progress; then
        log "INFO  | Upload complete: ${S3_BUCKET}/${S3_KEY}"
    else
        log_exit 3 "FAIL  | S3 upload failed for ${BACKUP_FILE}"
    fi
}

# ------------------------------------------------------------------------------
# Remove local backup file
# ------------------------------------------------------------------------------
cleanup_local() {
    rm -f "${BACKUP_FILE}"
    log "INFO  | Local file removed: ${BACKUP_FILE}"
}

# ------------------------------------------------------------------------------
# Enforce S3 retention (delete objects older than RETENTION_DAYS)
# ------------------------------------------------------------------------------
enforce_s3_retention() {
    log "INFO  | Enforcing S3 retention policy (${RETENTION_DAYS} days)"

    local cutoff_epoch
    cutoff_epoch=$(date -d "-${RETENTION_DAYS} days" +%s)

    local deleted=0

    while IFS= read -r line; do
        local obj_date obj_key obj_epoch
        obj_date=$(echo "${line}" | awk '{print $1}')
        obj_key=$(echo "${line}"  | awk '{print $4}')

        [[ -z "${obj_key}" ]] && continue

        obj_epoch=$(date -d "${obj_date}" +%s 2>/dev/null || true)

        if [[ -n "${obj_epoch}" ]] && (( obj_epoch < cutoff_epoch )); then
            if aws s3 rm "${S3_BUCKET}/${obj_key}" --quiet; then
                log "INFO  | Deleted expired object: ${obj_key} (date: ${obj_date})"
                (( deleted++ )) || true
            else
                log "WARN  | Failed to delete: ${obj_key}"
            fi
        fi
    done < <(aws s3 ls "${S3_BUCKET}/${DB_NAME}/" 2>/dev/null || true)

    log "INFO  | Retention enforcement complete (${deleted} object(s) deleted)"
}

# ------------------------------------------------------------------------------
# Main
# ------------------------------------------------------------------------------
main() {
    log "=========================================="
    log "INFO  | Backup job started"

    preflight_checks
    run_backup
    upload_to_s3
    cleanup_local
    enforce_s3_retention

    log "INFO  | Backup job finished successfully"
    log "=========================================="
}

main
```
```
# ==============================================================================
# Agendamento diário do backup às 22h — cron (Ubuntu 22.04)
# ==============================================================================

# 1. Copie o script para o local definitivo e ajuste as permissões
sudo cp ledger_backup.sh /usr/local/bin/ledger_backup.sh
sudo chmod 750 /usr/local/bin/ledger_backup.sh
sudo chown root:root /usr/local/bin/ledger_backup.sh

# 2. Garanta que o arquivo de log existe e tem permissão de escrita
sudo touch /var/log/ledger-backup.log
sudo chmod 640 /var/log/ledger-backup.log

# 3. Adicione a entrada no crontab do root
#    (o PGPASSWORD é injetado pela IAM role via Secrets Manager no ambiente de sistema;
#     se necessário, exporte-o explicitamente no cron conforme abaixo)
sudo crontab -e

# --- conteúdo a inserir no crontab ---
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Backup diário do PostgreSQL ledger_prod às 22h00 (horário UTC da instância)
0 22 * * * /usr/local/bin/ledger_backup.sh >> /var/log/ledger-backup.log 2>&1
# --- fim do conteúdo ---

# 4. Verifique se a entrada foi registrada corretamente
sudo crontab -l

# ------------------------------------------------------------------------------
# Notas
# ------------------------------------------------------------------------------
# • A IAM role da instância deve ter as policies: 
#     - AmazonS3FullAccess (ou policy customizada limitada ao bucket hvt-ledger-backups)
#     - secretsmanager:GetSecretValue (para o segredo que contém PGPASSWORD)
#
# • Se o PGPASSWORD não for injetado automaticamente no ambiente do cron,
#   adicione antes do comando no crontab:
#     0 22 * * * export PGPASSWORD=$(aws secretsmanager get-secret-value \
#       --secret-id <SECRET_ID> --query SecretString --output text | \
#       python3 -c "import sys,json;print(json.load(sys.stdin)['password'])") && \
#       /usr/local/bin/ledger_backup.sh >> /var/log/ledger-backup.log 2>&1
#
# • Para rotacionar o próprio log, crie /etc/logrotate.d/ledger-backup:
#     /var/log/ledger-backup.log {
#         weekly
#         rotate 8
#         compress
#         missingok
#         notifempty
#     }
```
---

## Justificativa
```
A Role define a persona técnica e o nível de senioridade (SRE especialista em Linux, PostgreSQL e AWS); a Task delimita a ação prática a ser executada com todos os seus requisitos técnicos e variáveis (desenvolver o script de backup e retenção); e o Format dita a estrutura final da resposta (apenas o código e o agendamento cron), restringindo textualmente qualquer tipo de explicação adicional.
```
