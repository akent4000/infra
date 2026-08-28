# Целевая архитектура

## 1. Главный принцип

Инфраструктура должна быть **воспроизводимой**, а не просто автоматизированной.

Автоматизация может выполнять ручные команды быстрее. Воспроизводимость означает, что desired state описан явно, и система может быть восстановлена из источников правды.

Разделяем:

```text
CONFIGURATION
Git / Ansible / Docker Compose / Caddy

SECRETS
Infisical + отдельный recovery store

DATA
PostgreSQL / MinIO / CouchDB / user files
        ↓
external S3 backups
```

## 2. Control plane и workload plane

Целевая архитектура:

```text
                         Internet
                            │
                      DNS provider
                            │
              ┌─────────────┴─────────────┐
              │                           │
           Caddy                      Warpgate
              │                           │
       public services              SSH gateway
                                          │
                                     Semaphore
                                          │
                                       Ansible
                                          │
                      ┌───────────────────┼───────────────┐
                      ▼                   ▼               ▼
                   app-01              vpn-01          app-02
                   Docker              Docker          Docker
                      │                   │               │
                      └───────────────────┼───────────────┘
                                          │
                                       backups
                                          │
                                          ▼
                                    External S3
```

## 3. Management plane

В него входят:

- Warpgate;
- Semaphore;
- Infisical;
- monitoring controller / alerting.

На текущем этапе допустимо держать их на одном хосте с приложениями, но долгосрочно лучше вынести management plane на отдельный маленький VPS или VM.

Пример:

```text
mgmt-01
├── Warpgate
├── Semaphore
├── Infisical
└── monitoring
```

## 4. Workload plane

Обычные серверы должны быть максимально скучными:

```text
Ubuntu
Docker
Docker Compose
backup agent
monitoring agent
Ansible user
Warpgate access
```

На уровне workload не нужен Kubernetes, если нет реальной потребности в scheduler/self-healing/cluster orchestration.

## 5. Типы серверов

Вместо `server1/server2/server3` лучше мыслить ролями:

```text
management
application
vpn
storage
development
```

Inventory затем группирует hosts по функциям.

## 6. Сетевой подход

Желаемая модель:

```text
PUBLIC
- 443 -> Caddy
- 2222 -> Warpgate
- VPN ports -> если нужны

PRIVATE
- Ansible
- database access
- internal APIs
- monitoring traffic
```

В будущем management nodes и workload nodes можно связать WireGuard/private network.

## 7. Warpgate

Warpgate — это access plane, а не система автоматизации.

Лучше разделять identities:

```text
human user
  ↓
Warpgate
  ↓
personal OS user

ansible-automation
  ↓
Warpgate
  ↓
ansible OS user
  ↓
sudo
```

Не стоит использовать один и тот же root target для людей и автоматизации.

## 8. Semaphore

Semaphore должен быть UI над Ansible, а не source of truth.

Правильная цепочка:

```text
Git
 ↓
Semaphore
 ↓
Ansible
 ↓
servers
```

Semaphore хранит inventory bindings, credentials, task templates и logs, но сами playbooks и roles живут в Git.

## 9. Infisical

Infisical — runtime secrets plane.

Секреты не должны попадать в Git.

Однако bootstrap secrets самого Infisical и credentials восстановления должны существовать отдельно:

```text
RECOVERY
├── Infisical bootstrap secrets
├── S3 recovery credentials
├── DNS / registrar recovery
├── Warpgate recovery
└── emergency SSH key
```

## 10. Backup plane

External S3 подходит как основной off-host backup target.

Рекомендуемая схема:

```text
Postgres -> pg_dump -> restic -> S3
MinIO/files -> restic -> S3
CouchDB -> consistent backup -> restic -> S3
Warpgate/Infisical data -> backup -> S3
```

Желательно:

- client-side encryption;
- versioning;
- object lock/immutability при наличии;
- отдельные credentials;
- retention policy;
- периодический restore test.

Backup считается реальным только после успешного восстановления на чистую VM.

## 11. Deployment flow

Нормальный путь изменений:

```text
Git commit
    ↓
Semaphore / CI
    ↓
Ansible
    ↓
server
    ├── copy config
    ├── obtain secrets
    ├── docker compose pull
    ├── docker compose up -d
    └── health check
```

Portainer лучше оставить как inspection/debug/emergency UI, а не как основной deployment path.

## 12. Desired state

Хороший Ansible описывает состояние:

```text
user exists
package installed
file has exact content
service enabled
firewall rule exists
```

а не просто автоматизирует shell-команды.

Ключевое свойство — **idempotency**: повторный запуск не должен ломать систему и не должен создавать ненужные изменения.

## 13. Disaster recovery

Желаемый сценарий:

```text
1. Provision clean Ubuntu
2. Add host to inventory
3. Run bootstrap.yml
4. Restore management stack
5. Restore Infisical
6. Deploy services
7. Restore application data from S3
8. Update DNS
9. Run health checks
```

Если это можно выполнить без ручного вспоминания старых правок — архитектура считается воспроизводимой.
