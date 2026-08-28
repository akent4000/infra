# host_vars for the test inventory

This directory holds per-host variables. Files matching `test-*.yml`
(except this README) are **not tracked in git** — they contain the real
`ansible_host` IP and any keys. The git-exclusion is configured via
`.git/info/exclude`.

Tracked example pattern (create as `test-01.yml`, untracked):

```yaml
---
ansible_host: 192.168.x.x
ansible_user: ubuntu
```
