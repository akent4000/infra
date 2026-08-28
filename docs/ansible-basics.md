# Ansible: inventories, playbooks и roles

## 1. Что делает Ansible

Ansible настраивает уже существующие серверы и приводит их к желаемому состоянию.

Пример:

```text
у меня уже есть Ubuntu server
        ↓
Ansible
        ↓
users
SSH
UFW
Docker
Caddy
backup agent
monitoring
```

## 2. Inventory

**Inventory** отвечает на вопрос:

> Какие у меня есть серверы и к каким группам они относятся?

Пример:

```yaml
all:
  children:
    management:
      hosts:
        mgmt-01:
          ansible_host: 10.0.0.10

    apps:
      hosts:
        app-01:
          ansible_host: 10.0.0.20
        app-02:
          ansible_host: 10.0.0.21
```

Можно использовать:

- `group_vars/` — общие переменные группы;
- `host_vars/` — переменные конкретного сервера.

Пример:

```yaml
# group_vars/all.yml
timezone: Europe/Riga

base_packages:
  - curl
  - git
  - htop
  - jq
```

## 3. Playbook

**Playbook** отвечает на вопрос:

> Что нужно применить и к каким hosts?

Пример:

```yaml
---
- name: Prepare application servers
  hosts: apps
  become: true

  roles:
    - base
    - docker
    - monitoring
```

Playbook — это основная точка запуска.

Полезные playbooks:

```text
bootstrap.yml
configure.yml
deploy.yml
update.yml
backup.yml
restore.yml
```

## 4. Role

**Role** — переиспользуемый модуль конфигурации.

Примеры:

```text
roles/base/
roles/users/
roles/ssh/
roles/ufw/
roles/docker/
roles/caddy/
roles/monitoring/
roles/backup/
roles/compose_service/
```

Типовая структура:

```text
roles/docker/
├── defaults/
│   └── main.yml
├── handlers/
│   └── main.yml
└── tasks/
    └── main.yml
```

## 5. Как они связаны

```text
Inventory
    ↓
какие серверы?

Playbook
    ↓
что на них применить?

Roles
    ↓
как привести конкретную часть системы
к нужному состоянию?
```

Пример:

```text
inventory:
app-01
app-02
    │
    ▼
bootstrap.yml
    │
    ├── base
    ├── users
    ├── ssh
    ├── ufw
    ├── docker
    └── backup
```

## 6. Bootstrap vs Deploy

Лучше разделять:

### `bootstrap.yml`

```text
clean Ubuntu
    ↓
managed infrastructure host
```

Делает:

- base packages;
- users;
- sudo;
- SSH;
- firewall;
- Docker;
- directories;
- monitoring;
- backup agent;
- Warpgate access.

### `deploy.yml`

```text
managed host
    ↓
applications
```

Это уменьшает blast radius: обновление приложения не должно неожиданно менять SSH или UFW.

## 7. Idempotency

Ansible должен описывать state, а не последовательность действий.

Плохо:

```yaml
- shell: |
    echo ... >> /etc/file
    sed ...
    docker ...
```

Лучше использовать Ansible modules:

```yaml
- name: Ensure Docker is running
  ansible.builtin.service:
    name: docker
    state: started
    enabled: true
```

Повторный запуск должен приводить к тому же состоянию.

## 8. Минимальный первый playbook

```yaml
---
- name: Test managed hosts
  hosts: all
  gather_facts: false

  tasks:
    - name: Ping
      ansible.builtin.ping:
```

Запуск:

```bash
ansible-playbook \
  -i ansible/inventories/production/hosts.yml \
  ansible/playbooks/ping.yml
```

Первая цель — получить успешный `pong`, и только потом писать сложные roles.
