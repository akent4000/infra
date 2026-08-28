# Инфраструктура: целевая архитектура и план развития

Этот набор документов фиксирует целевую модель инфраструктуры, основные принципы, роли инструментов и порядок миграции от текущего single-node сервера к воспроизводимой инфраструктуре.

## Главная цель

Любой сервер должен считаться расходным runtime-узлом.

Если сервер потерян, его должно быть возможно заменить новым чистым Ubuntu-хостом и восстановить инфраструктуру из трех источников:

1. **Git** — конфигурация и desired state.
2. **Infisical / recovery store** — секреты.
3. **External S3** — резервные копии данных.

Целевая модель:

```text
Git + Secrets + Backup
          │
          ▼
      clean Ubuntu
          │
          ▼
       Ansible
          │
          ▼
    working server
```

Критерий зрелости:

> Сервер можно удалить и восстановить эквивалентный с нуля, не вспоминая, что когда-то вручную менялось в `/etc`, `/opt`, UFW или Docker.

## Основные компоненты

- **OpenTofu** — создает и управляет внешней инфраструктурой через API: DNS, VPS, buckets, cloud firewall и т.д.
- **Ansible** — приводит уже существующие серверы к нужному состоянию.
- **Docker Compose** — описывает приложения на серверах.
- **Semaphore** — UI и оркестратор поверх Ansible.
- **Warpgate** — централизованный SSH bastion / access plane.
- **Infisical** — runtime secrets.
- **External S3** — off-host backup target.
- **Caddy** — reverse proxy / TLS ingress.
- **Netdata + external checks** — observability и alerting.

## Разделение ответственности

```text
tofu/
"что существует СНАРУЖИ серверов"

ansible/
"как устроены САМИ серверы"

services/
"что работает НА серверах"

Infisical
"секретные значения"

S3
"данные и backup"
```

## Рекомендуемая структура Git-репозитория

```text
infra/
├── README.md
├── .gitignore
│
├── tofu/
│   ├── cloudflare/
│   ├── compute/
│   └── storage/
│
├── ansible/
│   ├── ansible.cfg
│   ├── inventories/
│   ├── playbooks/
│   └── roles/
│
├── services/
│   ├── caddy/
│   ├── vaultwarden/
│   ├── semaphore/
│   ├── warpgate/
│   ├── infisical/
│   ├── remnawave/
│   ├── mida/
│   └── ...
│
├── backup/
│   ├── policies.yml
│   └── scripts/
│
├── docs/
│   ├── architecture.md
│   ├── ansible-basics.md
│   ├── tofu-cloudflare.md
│   ├── migration-plan.md
│   ├── disaster-recovery.md
│   └── networking.md
│
└── scripts/
```

## Что не должно попадать в Git

Никогда не хранить в репозитории:

- `.env` с реальными значениями;
- API tokens;
- private SSH keys;
- Infisical Universal Auth credentials;
- S3 secret keys;
- database dumps;
- backup archives;
- TLS private keys;
- реальные пароли.

## Рекомендуемый `.gitignore`

```gitignore
# Secrets
.env
.env.*
!.env.example

*.pem
*.key
id_rsa
id_ed25519

# Ansible
*.retry
.vault_pass

# Terraform / OpenTofu
*.tfstate
*.tfstate.*
.terraform/

# Backups
*.sql
*.sql.gz
*.dump
*.tar
*.tar.gz

# Editors / OS
.DS_Store
.idea/
.vscode/
```

## Краткая дорожная карта

1. Создать private repo `infra`.
2. Добавить Ansible inventory и `ping.yml`.
3. Сделать базовые roles: `base`, `users`, `ssh`, `ufw`, `docker`.
4. Подключить новый тестовый сервер/VM.
5. Добиться полного bootstrap одной командой.
6. Настроить external S3 + restic/Kopia и тест restore.
7. Подключить Semaphore к repo.
8. Перевести Caddy и Compose в Git как source of truth.
9. Добавить OpenTofu для Cloudflare DNS.
10. Только после этого переносить production workloads и расширять автоматизацию.
