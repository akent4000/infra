# Disaster Recovery

Этот документ должен со временем превратиться из концепции в точную пошаговую runbook-инструкцию.

## Что должно быть доступно вне основного сервера

- доступ к DNS provider / registrar;
- Cloudflare recovery credentials;
- emergency SSH key;
- Infisical bootstrap secrets;
- S3 recovery credentials;
- Git repository;
- инструкции по восстановлению;
- при необходимости offline encrypted copy критичных bootstrap secrets.

## Recovery flow

```text
1. Provision new Ubuntu host
2. Add host to Ansible inventory
3. Run bootstrap.yml
4. Install Docker / firewall / users / monitoring / backup agent
5. Restore management components
6. Restore Infisical
7. Retrieve service secrets
8. Deploy Docker Compose applications
9. Restore PostgreSQL / MinIO / CouchDB / user data from S3
10. Update DNS if IP changed
11. Run health checks
12. Verify backup jobs on new host
```

## Restore priorities

### Tier 0 — access and secrets

- SSH / Warpgate recovery;
- Infisical;
- DNS access;
- S3 access.

### Tier 1 — management

- Semaphore;
- monitoring;
- Caddy if needed for management UI.

### Tier 2 — critical applications

- Vaultwarden;
- MIDA backend/data;
- Remnawave;
- other production services.

### Tier 3 — convenience services

- Jellyfin;
- MeTube;
- noncritical personal tools.

## Что считать успешным restore test

Backup нельзя считать проверенным только потому, что файл существует в S3.

Успешный тест означает:

1. создана чистая VM;
2. на ней восстановлен сервис;
3. сервис запускается;
4. данные читаются приложением;
5. health check проходит;
6. процесс документирован.

## RPO / RTO

Позже следует зафиксировать:

- **RPO** — сколько данных допустимо потерять;
- **RTO** — сколько времени допустимо восстанавливать сервис.

Для каждого критичного сервиса значения могут быть разными.
