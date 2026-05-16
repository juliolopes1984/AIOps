## Prompt:
```
# Context
Trabalhamos com terraform na AWS. Seguimos o padrão de módulos do terraform-aws-modules como referência de estrutura. Backend em S3.

# Action
Crie um terraform module reutilizável para provisionar buckets S3, com versionamento, encryption habilitada SSE-S3 mínimo, acesso público totalmente bloqueado, logging configurado, com as tags obrigatórias Owner, CostCenter e Environment

# Result
O módulo deve conter main.tf, backend.tf, variables.tf com description e type obrigatórios, output.tf retornando o ID do recurso criado e um README.md com descrição do módulo, orientando o uso do prefixo "hvt-" no nome dos recursos e exemplo de utilização do módulo.

# Example
Aqui está o exemplo de um módulo que está em produção, utilize-o como modelo para criação do novo módulo:

variable "environment" {
  description = "Nome do ambiente (dev, staging, production)"
  type        = string
}

locals {
  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }
}

resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
  tags = merge(local.common_tags, {
    Name = "hvt-vpc-${var.environment}"
  })
}

```
---

## Modelo:
```
GPT-5.5 e Opus 4.7
```
---

## Output GPT-5.5:
```
Abaixo está um módulo Terraform reutilizável para criação de buckets S3 seguindo suas diretrizes (boas práticas, segurança e padrão de tags).
---

# 📦 Estrutura do módulo

---
s3-bucket-module/
├── main.tf
├── backend.tf
├── variables.tf
├── outputs.tf
└── README.md
---

## 🧩 main.tf

hcl
---
locals {
  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }
}

resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name

  tags = merge(local.common_tags, {
    Name = var.bucket_name
  })
}

# Versioning
resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Server-side encryption (SSE-S3)
resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# Block all public access
resource "aws_s3_bucket_public_access_block" "this" {
  bucket = aws_s3_bucket.this.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Logging
resource "aws_s3_bucket_logging" "this" {
  bucket = aws_s3_bucket.this.id

  target_bucket = var.logging_bucket
  target_prefix = "${var.bucket_name}/"
}
---

## ⚙️ backend.tf

hcl
---
terraform {
  required_version = ">= 1.0"

  backend "s3" {
    bucket = "hvt-terraform-state"
    key    = "s3-bucket-module/terraform.tfstate"
    region = "us-east-1"
  }
}
---

## 🧾 variables.tf

hcl
---
variable "bucket_name" {
  description = "Nome do bucket S3 (usar prefixo hvt-)"
  type        = string
}

variable "environment" {
  description = "Nome do ambiente (dev, staging, production)"
  type        = string
}

variable "owner" {
  description = "Responsável pelo recurso"
  type        = string
}

variable "cost_center" {
  description = "Centro de custo do recurso"
  type        = string
}

variable "logging_bucket" {
  description = "Bucket de destino para armazenar logs de acesso"
  type        = string
}
---

---

## 📤 outputs.tf

hcl
---
output "bucket_id" {
  description = "ID do bucket S3 criado"
  value       = aws_s3_bucket.this.id
}
---

---

## 📘 README.md

md
---
# S3 Bucket Module

Módulo Terraform reutilizável para criação de buckets S3 seguindo boas práticas de segurança e governança.

## 🔒 Features

- Versionamento habilitado
- Criptografia SSE-S3 (AES256)
- Bloqueio total de acesso público
- Logging de acesso habilitado
- Tags obrigatórias:
  - Owner
  - CostCenter
  - Environment

## 📛 Padrão de nomenclatura

Todos os buckets devem utilizar o prefixo:

---
hvt-
---

Exemplo:
---
hvt-app-logs-dev
---

## 🚀 Exemplo de uso

hcl
---
module "s3_bucket" {
  source = "./s3-bucket-module"

  bucket_name    = "hvt-app-logs-dev"
  environment    = "dev"
  owner          = "plataforma"
  cost_center    = "cc-1234"
  logging_bucket = "hvt-central-logs"
}
---

## 📤 Outputs

| Nome      | Descrição           |
| --------- | ------------------- |
| bucket_id | ID do bucket criado |


---

Se quiser, posso evoluir esse módulo com:
- :contentReference[oaicite:0]{index=0}
- :contentReference[oaicite:1]{index=1}
- :contentReference[oaicite:2]{index=2}
- :contentReference[oaicite:3]{index=3}

Só dizer 👍
```

## Output Opus 4.7:
```
Vou criar um módulo Terraform reutilizável para buckets S3 seguindo o padrão que você forneceu. Vou estruturar os arquivos no diretório de trabalho e disponibilizá-los para download.Módulo `terraform-aws-s3-hvt` pronto. Resumo das decisões tomadas:

- **Nomenclatura `hvt-`**: aplicada automaticamente no `local.bucket_name` como `hvt-<bucket_name>-<environment>`, seguindo o mesmo padrão do exemplo em produção que você passou (`hvt-vpc-${var.environment}`). Isso evita que quem consome o módulo precise lembrar de adicionar o prefixo manualmente.
- **Encryption SSE-S3 mínimo**: default `AES256`, mas o módulo aceita `aws:kms` opcionalmente com `kms_master_key_id` — atende o requisito mínimo sem fechar a porta para algo mais forte.
- **Bloqueio público total**: as quatro flags do `aws_s3_bucket_public_access_block` são forçadas a `true` e não foram expostas como variáveis, justamente para impedir override acidental.
- **Logging**: `logging_target_bucket` é obrigatório (sem default) para não permitir um bucket ser criado sem destino de logs configurado. O prefix tem fallback para `logs/<bucket>/`.
- **Tags obrigatórias**: `Owner`, `CostCenter`, `Environment` vêm das variáveis e são mescladas em `local.common_tags`, exatamente no padrão do seu exemplo.
- **Backend S3**: configurado como esqueleto em `backend.tf` com valores placeholder, esperando override via `-backend-config` no `init` (esse é o padrão recomendado, já que valores hardcoded de backend não aceitam interpolação).

Um ponto que vale validar com seu time: deixei `environment` com validação restrita a `dev`, `staging`, `production`. Se vocês usam outros nomes (ex.: `prd`, `hml`, `sandbox`), ajustem a lista no bloco `validation` de `variables.tf`.

main.tf
---
locals {
  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }

  bucket_name = "hvt-${var.bucket_name}-${var.environment}"
}

resource "aws_s3_bucket" "this" {
  bucket = local.bucket_name

  force_destroy = var.force_destroy

  tags = merge(local.common_tags, var.additional_tags, {
    Name = local.bucket_name
  })
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id

  versioning_configuration {
    status = var.versioning_enabled ? "Enabled" : "Suspended"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = var.sse_algorithm
      kms_master_key_id = var.sse_algorithm == "aws:kms" ? var.kms_master_key_id : null
    }
    bucket_key_enabled = var.sse_algorithm == "aws:kms" ? true : false
  }
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket = aws_s3_bucket.this.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_logging" "this" {
  bucket = aws_s3_bucket.this.id

  target_bucket = var.logging_target_bucket
  target_prefix = var.logging_target_prefix != "" ? var.logging_target_prefix : "logs/${local.bucket_name}/"
}
---

variables.tf
---
variable "bucket_name" {
  description = "Nome do bucket S3 (sem o prefixo 'hvt-' e sem o sufixo de ambiente, ambos adicionados automaticamente pelo módulo). Ex.: 'app-logs' resultará em 'hvt-app-logs-<environment>'."
  type        = string
}

variable "environment" {
  description = "Nome do ambiente (dev, staging, production)."
  type        = string

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "O valor de 'environment' deve ser um dos seguintes: dev, staging, production."
  }
}

variable "owner" {
  description = "Time ou pessoa responsável pelo recurso. Aplicado como tag obrigatória 'Owner'."
  type        = string
}

variable "cost_center" {
  description = "Centro de custo associado ao recurso. Aplicado como tag obrigatória 'CostCenter'."
  type        = string
}

variable "versioning_enabled" {
  description = "Habilita versionamento no bucket S3. Recomendado mantê-lo como true em produção."
  type        = bool
  default     = true
}

variable "sse_algorithm" {
  description = "Algoritmo de criptografia server-side. Aceita 'AES256' (SSE-S3) ou 'aws:kms' (SSE-KMS). Mínimo exigido é SSE-S3."
  type        = string
  default     = "AES256"

  validation {
    condition     = contains(["AES256", "aws:kms"], var.sse_algorithm)
    error_message = "O valor de 'sse_algorithm' deve ser 'AES256' ou 'aws:kms'."
  }
}

variable "kms_master_key_id" {
  description = "ARN ou ID da chave KMS usada quando sse_algorithm = 'aws:kms'. Ignorado se sse_algorithm for 'AES256'."
  type        = string
  default     = null
}

variable "logging_target_bucket" {
  description = "Nome do bucket S3 de destino para armazenamento dos logs de acesso. Deve existir previamente e estar na mesma região."
  type        = string
}

variable "logging_target_prefix" {
  description = "Prefixo (path) usado dentro do bucket de logging. Se vazio, o módulo usa 'logs/<nome-do-bucket>/'."
  type        = string
  default     = ""
}

variable "force_destroy" {
  description = "Permite destruição do bucket mesmo contendo objetos. Use com cautela; mantenha como false em produção."
  type        = bool
  default     = false
}

variable "additional_tags" {
  description = "Tags adicionais que serão mescladas às tags obrigatórias (Owner, CostCenter, Environment, Name)."
  type        = map(string)
  default     = {}
}
---

output.tf
---
output "id" {
  description = "ID (nome) do bucket S3 criado."
  value       = aws_s3_bucket.this.id
}

output "arn" {
  description = "ARN do bucket S3 criado."
  value       = aws_s3_bucket.this.arn
}

output "bucket_domain_name" {
  description = "Nome de domínio do bucket S3 (formato: <bucket>.s3.amazonaws.com)."
  value       = aws_s3_bucket.this.bucket_domain_name
}

output "bucket_regional_domain_name" {
  description = "Nome de domínio regional do bucket S3."
  value       = aws_s3_bucket.this.bucket_regional_domain_name
}
---

backend.tf
---
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }

  backend "s3" {
    # Os valores abaixo devem ser sobrescritos via -backend-config
    # ou em um arquivo backend.hcl no momento do terraform init.
    #
    # Exemplo:
    #   terraform init \
    #     -backend-config="bucket=hvt-tfstate-<env>" \
    #     -backend-config="key=s3/<bucket_name>/terraform.tfstate" \
    #     -backend-config="region=us-east-1" \
    #     -backend-config="dynamodb_table=hvt-tfstate-lock" \
    #     -backend-config="encrypt=true"
    bucket         = "hvt-tfstate"
    key            = "s3/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "hvt-tfstate-lock"
    encrypt        = true
  }
}
---

README.md
---
# terraform-aws-s3-hvt

Módulo Terraform para provisionamento de buckets S3 na AWS seguindo o padrão de módulos do [terraform-aws-modules](https://github.com/terraform-aws-modules) e as práticas internas da organização.

## Descrição

Este módulo cria um bucket S3 com a configuração mínima de segurança exigida pela organização:

- **Versionamento** habilitado por padrão (configurável).
- **Criptografia server-side** habilitada — mínimo `SSE-S3` (`AES256`); permite optar por `SSE-KMS` informando uma chave.
- **Bloqueio total de acesso público** (`block_public_acls`, `block_public_policy`, `ignore_public_acls`, `restrict_public_buckets`).
- **Logging de acesso** configurado em um bucket de destino informado pela pessoa que consome o módulo.
- **Tags obrigatórias** aplicadas automaticamente: `Owner`, `CostCenter`, `Environment` e `Name`.

## Convenção de nomenclatura

Todos os recursos provisionados na organização devem usar o prefixo **`hvt-`**. O módulo já adiciona esse prefixo automaticamente, além do sufixo de ambiente, ao valor de `bucket_name`:

---
hvt-<bucket_name>-<environment>
---

Exemplo: passando `bucket_name = "app-logs"` e `environment = "production"`, o bucket criado terá o nome `hvt-app-logs-production`. **Não inclua `hvt-` manualmente no valor de `bucket_name`** — isso evita duplicação.

## Backend

O backend é configurado em `backend.tf` para uso de S3 + DynamoDB para lock. Os valores devem ser sobrescritos no `terraform init` via `-backend-config` ou arquivo `backend.hcl`:

---
terraform init \
  -backend-config="bucket=hvt-tfstate-production" \
  -backend-config="key=s3/app-logs/terraform.tfstate" \
  -backend-config="region=us-east-1" \
  -backend-config="dynamodb_table=hvt-tfstate-lock" \
  -backend-config="encrypt=true"
---

## Exemplo de uso

---
module "s3_app_logs" {
  source = "git::ssh://git@github.com/hvt/terraform-aws-s3-hvt.git?ref=v1.0.0"

  bucket_name = "app-logs"
  environment = "production"

  owner       = "plataforma"
  cost_center = "CC-1042"

  # Bucket destino para logs de acesso (deve existir previamente)
  logging_target_bucket = "hvt-s3-access-logs-production"
  logging_target_prefix = "logs/app-logs/"

  versioning_enabled = true
  sse_algorithm      = "AES256"

  additional_tags = {
    Application = "checkout-api"
  }
}

output "app_logs_bucket_id" {
  value = module.s3_app_logs.id
}
---

### Exemplo com SSE-KMS

---
module "s3_sensitive_data" {
  source = "git::ssh://git@github.com/hvt/terraform-aws-s3-hvt.git?ref=v1.0.0"

  bucket_name = "sensitive-data"
  environment = "production"

  owner       = "seguranca"
  cost_center = "CC-2001"

  logging_target_bucket = "hvt-s3-access-logs-production"

  sse_algorithm     = "aws:kms"
  kms_master_key_id = "arn:aws:kms:us-east-1:123456789012:key/abcd-1234-..."
}
---

## Inputs

| Nome | Descrição | Tipo | Default | Obrigatório |
|---|---|---|---|:---:|
| `bucket_name` | Nome do bucket S3 (sem prefixo `hvt-` e sem sufixo de ambiente). | `string` | n/a | sim |
| `environment` | Ambiente: `dev`, `staging` ou `production`. | `string` | n/a | sim |
| `owner` | Time ou pessoa responsável. Tag `Owner`. | `string` | n/a | sim |
| `cost_center` | Centro de custo. Tag `CostCenter`. | `string` | n/a | sim |
| `logging_target_bucket` | Bucket destino para logs de acesso. | `string` | n/a | sim |
| `logging_target_prefix` | Prefixo dentro do bucket de logging. | `string` | `""` | não |
| `versioning_enabled` | Habilita versionamento. | `bool` | `true` | não |
| `sse_algorithm` | `AES256` (SSE-S3) ou `aws:kms` (SSE-KMS). | `string` | `"AES256"` | não |
| `kms_master_key_id` | Chave KMS quando `sse_algorithm = "aws:kms"`. | `string` | `null` | não |
| `force_destroy` | Permite destruir bucket com objetos. | `bool` | `false` | não |
| `additional_tags` | Tags adicionais mescladas às obrigatórias. | `map(string)` | `{}` | não |

## Outputs

| Nome | Descrição |
|---|---|
| `id` | ID (nome) do bucket S3 criado. |
| `arn` | ARN do bucket. |
| `bucket_domain_name` | Domain name do bucket. |
| `bucket_regional_domain_name` | Domain name regional do bucket. |

## Requisitos

| Nome | Versão |
|---|---|
| terraform | `>= 1.5.0` |
| aws provider | `>= 5.0` |
---
```
---

## Justificativa:
```
O Context define o Terraform como ferramenta padrão de provisionamento de código e o S# como backend de armazenamento do tfstate; o Action orienta a criação de um módulo Terraform reutilizável para provisionamento de Bucket S3 com os requisitos de segurança e utilizando as melhores práticas definidas pela companhia; o Result especifica todos os arteatos que precisam ser criados e as instruções a serem incluídas no README; o Example fornece um modelo que já é tuilizado pela companhia para que o mesmo seja usado como modelo para a criação do novo módulo.
```
