# OpenTofu, Terraform и Cloudflare

## 1. Terraform

Terraform — Infrastructure as Code инструмент, который управляет внешними ресурсами через API провайдеров.

Если Ansible обычно работает так:

```text
сервер уже существует
      ↓
настроить его
```

то Terraform/OpenTofu:

```text
мне нужен ресурс
      ↓
создать / изменить / удалить его через API
```

Примеры ресурсов:

- VPS;
- cloud networks;
- firewalls;
- DNS records;
- S3 buckets;
- load balancers;
- GitHub repositories;
- IAM resources.

## 2. OpenTofu

OpenTofu — open-source альтернатива Terraform, созданная как fork после изменения лицензирования Terraform.

По модели и синтаксису они очень похожи.

Terraform:

```bash
terraform init
terraform plan
terraform apply
```

OpenTofu:

```bash
tofu init
tofu plan
tofu apply
```

Для новой собственной инфраструктуры OpenTofu — хороший выбор.

## 3. OpenTofu vs Ansible

Запомнить можно так:

> OpenTofu создаёт инфраструктуру. Ansible настраивает инфраструктуру.

Пример нового VPS:

```text
OpenTofu
├── create VPS
├── create cloud firewall
├── create Cloudflare DNS record
└── create S3 bucket

Ansible
├── create users
├── configure SSH
├── configure UFW
├── install Docker
├── install monitoring
└── deploy applications
```

## 4. Provider

OpenTofu сам не знает, как работать с Cloudflare, AWS или Hetzner.

Для этого используются **providers**.

```text
OpenTofu
   │
   ├── Cloudflare Provider -> Cloudflare API
   ├── AWS Provider -> AWS API
   ├── Hetzner Provider -> Hetzner API
   └── ...
```

## 5. Cloudflare + OpenTofu

Структура:

```text
infra/
└── tofu/
    └── cloudflare/
        ├── versions.tf
        ├── provider.tf
        ├── variables.tf
        ├── dns.tf
        └── outputs.tf
```

### `versions.tf`

```hcl
terraform {
  required_providers {
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 5"
    }
  }
}
```

Даже в OpenTofu блок называется `terraform {}` — это совместимый синтаксис.

### `provider.tf`

```hcl
provider "cloudflare" {
  api_token = var.cloudflare_api_token
}
```

### `variables.tf`

```hcl
variable "cloudflare_api_token" {
  type      = string
  sensitive = true
}

variable "cloudflare_zone_id" {
  type = string
}
```

### `dns.tf`

Пример DNS record:

```hcl
resource "cloudflare_dns_record" "vault" {
  zone_id = var.cloudflare_zone_id

  name    = "vault.akent.site"
  type    = "A"
  content = "1.2.3.4"

  proxied = true
  ttl     = 1
}
```

## 6. Credentials

Cloudflare API token нельзя хранить в Git.

Варианты:

```bash
export CLOUDFLARE_API_TOKEN="..."
```

или через Infisical:

```text
Infisical
   ↓
CLOUDFLARE_API_TOKEN
   ↓
OpenTofu
   ↓
Cloudflare API
```

## 7. `plan` и `apply`

`tofu plan` показывает, что изменится.

Пример:

```text
+ create cloudflare_dns_record.app
~ update cloudflare_dns_record.vault
- destroy cloudflare_dns_record.old
```

`tofu apply` применяет эти изменения.

Правильный workflow:

```text
edit .tf
  ↓
tofu plan
  ↓
review
  ↓
tofu apply
```

## 8. State

OpenTofu/Terraform хранит **state** — соответствие между кодом и реальными ресурсами.

```text
cloudflare_dns_record.vault
        ↕
Cloudflare object id=...
```

State нужен, чтобы понимать:

- какой ресурс уже существует;
- что изменилось;
- что нужно создать;
- что нужно удалить.

State нельзя бездумно коммитить в Git.

Позже лучше использовать remote state в защищенном backend / S3.

## 9. Уже существующие DNS записи

Если DNS records уже созданы вручную, нельзя просто описать их в `.tf` и сразу делать `apply`.

Нужно выполнить миграцию:

```text
Cloudflare UI records
        ↓
описать в .tf
        ↓
import existing resources into state
        ↓
tofu plan
        ↓
No changes
```

Только после `No changes` запись считается корректно перенесённой под IaC.

## 10. Почему DNS лучше не вести через Ansible

Технически можно, но граница ответственности чище такая:

```text
OpenTofu
"какие внешние ресурсы существуют?"

Ansible
"как настроены мои серверы?"

Docker Compose
"какие приложения работают?"

Infisical
"какие секреты используются?"

S3
"где лежат backup данные?"
```
