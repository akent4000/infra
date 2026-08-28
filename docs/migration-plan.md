# План миграции к воспроизводимой инфраструктуре

Главный принцип: **не делать big-bang migration**.

Сначала новое desired state должно уметь воспроизводить текущее рабочее состояние. Только потом можно менять архитектуру.

## Этап 0. Зафиксировать текущее состояние

Не менять production.

Сохранить:

- текущий `INFRA.md`;
- список Docker Compose проектов;
- Caddy config;
- UFW rules;
- systemd units;
- `/etc/hosts` особенности;
- backup locations;
- список доменов;
- список секретов по сервисам без самих значений.

## Этап 1. Создать private Git repo

Минимальная первая структура:

```text
infra/
├── README.md
├── .gitignore
├── ansible/
│   ├── ansible.cfg
│   ├── inventories/
│   │   └── production/
│   │       ├── hosts.yml
│   │       └── host_vars/
│   ├── playbooks/
│   │   └── ping.yml
│   └── roles/
├── services/
│   └── README.md
└── docs/
    ├── architecture.md
    └── current-infrastructure.md
```

Первый успех:

```bash
ansible-playbook -i ansible/inventories/production/hosts.yml ansible/playbooks/ping.yml
```

## Этап 2. Тестовая VM / второй сервер

Не начинать с production host.

Взять чистую Ubuntu VM/VPS и сделать её первым полностью Ansible-managed node.

Начать с roles:

```text
base
users
ssh
ufw
docker
monitoring
backup
```

Цель:

```bash
ansible-playbook bootstrap.yml --limit test-01
```

после чего test-01 полностью готов к использованию.

## Этап 3. Automation identity

Создать отдельного OS user:

```text
ansible
```

или:

```text
automation
```

Не использовать human account и не использовать root как основной automation identity.

Схема:

```text
Semaphore
   ↓
Warpgate
   ↓
ansible@server
   ↓
sudo
```

## Этап 4. Backup

Подключить external S3.

Использовать restic или Kopia.

Сначала бэкапировать тестовый сервер, затем production.

Для БД делать application-consistent backup:

```text
Postgres -> pg_dump -> restic -> S3
```

а не копировать живой `/var/lib/postgresql`.

Настроить retention, например:

```text
7 daily
4 weekly
12 monthly
```

Критический этап — restore test на отдельную VM.

## Этап 5. Semaphore

Подключить Git repo к Semaphore.

Создать task templates:

```text
Ping all
Server status
Bootstrap host
Update packages
Docker status
Deploy service
Backup
Restore test
```

Semaphore не становится source of truth: source of truth остается Git.

## Этап 6. Caddy в Git

Сначала просто перенести существующие рабочие конфиги без изменения логики.

```text
services/caddy/
├── docker-compose.yml
├── Caddyfile
└── sites/
```

После этого `/opt/caddy` становится deployed copy, а не source of truth.

## Этап 7. Docker Compose в Git

Переносить по одному сервису.

Пример:

```text
services/vaultwarden/
├── docker-compose.yml
├── README.md
└── caddy.caddy
```

Секреты не переносить — они остаются в Infisical.

Переходная схема может оставаться такой:

```text
Semaphore
 ↓
Ansible
 ↓
existing deploy.sh / deploy-envfile.sh
 ↓
Infisical
 ↓
Docker Compose
```

Позже shell scripts можно заменить универсальной Ansible role `compose_service`.

## Этап 8. OpenTofu + Cloudflare

Не трогать все DNS записи сразу.

Порядок:

1. установить OpenTofu;
2. создать `tofu/cloudflare`;
3. создать Cloudflare API token с минимальными правами;
4. выбрать одну некритичную DNS запись;
5. описать её в `.tf`;
6. import existing record into state;
7. добиться `tofu plan -> No changes`;
8. только затем переносить остальные records.

## Этап 9. Управление production host через Ansible

После того как тестовый узел стабильно воспроизводится, постепенно привести текущий production server к тому же desired state.

Порядок:

```text
base
users
Docker
SSH
UFW
Caddy
monitoring
backup
applications
```

Ansible сначала должен отражать существующее состояние, а не менять его.

## Этап 10. Разделение management и workloads

Когда появятся дополнительные узлы:

```text
mgmt-01
├── Warpgate
├── Semaphore
├── Infisical
└── monitoring

app-01
├── Docker workloads

vpn-01
├── VPN workloads
```

Так исчезает главный single-node failure domain.

## Этап 11. Disaster recovery exercise

Провести реальный тест:

```text
основной сервер считается потерянным
        ↓
новый чистый Ubuntu
        ↓
bootstrap
        ↓
restore control plane
        ↓
restore Infisical
        ↓
deploy apps
        ↓
restore data from S3
        ↓
DNS switch
        ↓
health checks
```

После успешного теста можно говорить, что инфраструктура действительно воспроизводима.

## Чего пока не добавлять

Не вводить без реальной необходимости:

- Kubernetes;
- Nomad;
- Consul;
- Vault;
- Ceph;
- Longhorn;
- ArgoCD;
- service mesh.

Сначала решить backup, restore, reproducibility и separation of concerns.
