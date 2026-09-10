# zulip-compose

[Zulip](https://zulip.com/) team chat for claimjumper.ai as a Docker Compose stack,
deployed git-ops style from this repo by Portainer onto the `apps` host.

Based on the official [zulip/docker-zulip](https://github.com/zulip/docker-zulip)
`compose.yaml` (Zulip 12.x): `zulip` (ghcr.io/zulip/zulip-server), `database`
(zulip/zulip-postgresql), `memcached`, `rabbitmq`, `redis`. All versions pinned, all
data in named volumes.

Differences from upstream:

- **No published ports.** The per-host Traefik stack
  ([traefik-compose](https://github.com/claimjumper-ai/traefik-compose)) owns 80/443.
  The `zulip` container joins the external `proxy` network and is routed by labels
  (`Host(zulip.claimjumper.ai)` → port 80, entrypoint `websecure`, certresolver `le`).
  `CERTIFICATES` stays unset so nginx inside the container serves plain HTTP;
  `LOADBALANCER_IPS` (default `172.16.0.0/12`, the Docker bridge range) makes Zulip
  trust `X-Forwarded-For` / `X-Forwarded-Proto` from Traefik.
- **Everything from the environment.** Settings are `SETTING_*` env vars, secrets are
  Compose secrets sourced from `ZULIP__*` env vars. Nothing secret is in git.

## Deployment (Portainer git-ops)

Portainer → environment `apps` → Stacks → Add stack → Repository:

| Field        | Value                                             |
| ------------ | ------------------------------------------------- |
| Name         | `zulip`                                           |
| Repository   | `https://github.com/claimjumper-ai/zulip-compose` |
| Reference    | `refs/heads/main`                                 |
| Compose path | `docker-compose.yml`                              |

Prerequisites: the Traefik stack is running on the host (network `proxy` exists) and
DNS for `ZULIP_HOST` points at it.

Environment variables (see [.env.example](.env.example)):

| Variable                        | Required | Notes                                          |
| ------------------------------- | -------- | ---------------------------------------------- |
| `ZULIP_HOST`                    | yes      | Public hostname, used for the Traefik rule too |
| `SETTING_ZULIP_ADMINISTRATOR`   | yes      | Admin contact shown to users and in error mail |
| `ZULIP__POSTGRES_PASSWORD`      | yes      | `openssl rand -hex 32`; only read on first boot |
| `ZULIP__MEMCACHED_PASSWORD`     | yes      | `openssl rand -hex 32`                         |
| `ZULIP__RABBITMQ_PASSWORD`      | yes      | `openssl rand -hex 32`                         |
| `ZULIP__REDIS_PASSWORD`         | yes      | `openssl rand -hex 32`                         |
| `ZULIP__SECRET_KEY`             | yes      | `openssl rand -hex 32`; Django secret key      |
| `SETTING_EMAIL_HOST`            | yes      | SMTP host                                      |
| `SETTING_EMAIL_HOST_USER`       | yes      | SMTP user                                      |
| `ZULIP__EMAIL_PASSWORD`         | yes      | SMTP password                                  |
| `SETTING_EMAIL_PORT`            | no       | default `587`                                  |
| `SETTING_EMAIL_USE_TLS`         | no       | default `True` (STARTTLS)                      |
| `SETTING_NOREPLY_EMAIL_ADDRESS` | yes      | Sender address for system mail                 |
| `LOADBALANCER_IPS`              | no       | default `172.16.0.0/12`                        |
| `QUEUE_WORKERS_MULTIPROCESS`    | no       | default `False` (saves RAM on the shared host) |

Updates: change the compose file, push to `main`, then "Pull and redeploy" the stack in
Portainer (or call its redeploy webhook). Edge deploys are asynchronous: check the
stack status or `docker ps` on the host.

Changing `ZULIP__POSTGRES_PASSWORD` after the first boot needs a manual `ALTER ROLE`
in the database, see the
[docker-zulip secrets docs](https://zulip.readthedocs.io/projects/docker/en/latest/how-to/compose-secrets.html#rotating-the-postgresql-password).

## First run

Zulip has no default admin. Generate a one-time organisation creation link on the host
and open it in a browser:

```bash
docker exec -u zulip zulip /home/zulip/deployments/current/manage.py generate_realm_creation_link
```

## Mobile push notifications (later)

Phones get no notifications until the server is registered with Zulip's push
notification service (free for small organisations). Add
`SETTING_ZULIP_SERVICE_PUSH_NOTIFICATIONS=True` to the stack environment, redeploy,
then run:

```bash
docker exec -u zulip zulip /home/zulip/deployments/current/manage.py register_server
```

Docs: <https://zulip.readthedocs.io/en/stable/production/mobile-push-notifications.html>

## Backups

- Volumes: `zulip_zulip` (uploads, settings, secrets), `zulip_postgresql-14`,
  `zulip_rabbitmq`, `zulip_redis`. The Hetzner disk backup covers them.
- The container takes a daily database dump into `/data/backups/` on the `zulip_zulip`
  volume (`AUTO_BACKUP_ENABLED`, on by default).
- Full consistent backup (database + uploads + settings) on demand:

  ```bash
  docker exec -u zulip zulip /home/zulip/deployments/current/manage.py backup --output=/data/backups/zulip-backup.tar.gz
  ```

  Restore: <https://zulip.readthedocs.io/en/stable/production/export-and-import.html#backups>

## Links

- docker-zulip docs: <https://zulip.readthedocs.io/projects/docker/en/latest/>
- Zulip server settings: <https://zulip.readthedocs.io/en/stable/production/settings.html>
