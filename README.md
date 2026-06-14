# netbox-ansible

Ansible role to deploy [NetBox](https://github.com/netbox-community/netbox) in Docker with support for plugins, automatic backups, and zero-downtime updates.

## Requirements

- Ansible Core 2.14+
- `community.docker` collection ≥ 3.10 (`ansible-galaxy collection install -r requirements.yml`)
- Target host: Docker with the Compose plugin (`docker compose`)
  - Supported OS: RHEL/Rocky/Alma 9, Debian 12, Ubuntu 22.04/24.04, Arch Linux
  - Docker is installed automatically if not present

## Quick start

```bash
# Install dependencies
ansible-galaxy collection install -r requirements.yml

# Edit inventory and secrets
vi inventory/hosts.yml
vi group_vars/netbox.yml

# Deploy
ansible-playbook playbook.yml
```

## Inventory

```yaml
# inventory/hosts.yml
all:
  children:
    netbox:
      hosts:
        netbox01:
          ansible_host: 192.168.1.10
          ansible_user: ansible
```

## Variables

All variables have defaults in `roles/netbox_docker/defaults/main.yml`. Override them in `group_vars/netbox.yml` or `host_vars/`.

### Core

| Variable | Default | Description |
|---|---|---|
| `netbox_version` | `v4.2` | Image tag from `netboxcommunity/netbox` |
| `netbox_base_dir` | `/opt/netbox` | Deploy directory on the target host |
| `netbox_http_port` | `8080` | Host port to expose NetBox on |
| `netbox_time_zone` | `UTC` | Django `TIME_ZONE` setting |
| `netbox_allowed_hosts` | `["*"]` | Django `ALLOWED_HOSTS` |
| `netbox_secret_key` | — | **Required.** Store in vault. |
| `netbox_compose_project` | `netbox` | Docker Compose project name |

### Superuser (first run only)

| Variable | Default |
|---|---|
| `netbox_superuser_name` | `admin` |
| `netbox_superuser_email` | `admin@example.com` |
| `netbox_superuser_password` | — **Store in vault.** |
| `netbox_superuser_api_token` | `""` (auto-generated) |

### Database

| Variable | Default |
|---|---|
| `netbox_db_name` | `netbox` |
| `netbox_db_user` | `netbox` |
| `netbox_db_password` | — **Store in vault.** |
| `netbox_db_host` | `postgres` |
| `netbox_db_port` | `5432` |

### Redis

Two Redis instances are deployed: one for task queues, one for caching.

| Variable | Default |
|---|---|
| `netbox_redis_host` | `redis` |
| `netbox_redis_password` | `""` |
| `netbox_redis_cache_host` | `redis-cache` |
| `netbox_redis_cache_password` | `""` |

### Plugins

```yaml
netbox_plugins:
  - pip: "netbox-custom-objects"       # pip package name
    name: "netbox_custom_objects"      # Python module name
    config: {}                         # plugin-specific settings

  - pip: "netbox-topology-views"
    name: "netbox_topology_views"
    config:
      allow_coordinates_saving: true
```

When `netbox_plugins` is non-empty the role builds a custom Docker image (`netbox_image_name:netbox_image_tag`) with plugins baked in via pip. Set `netbox_plugins: []` to use the upstream image directly.

Extra pip packages (not registered as plugins):

```yaml
netbox_extra_packages:
  - netbox-rest-api-haystack
```

### Non-root Docker access

```yaml
netbox_docker_users:
  - alice
  - deploy
```

Adds listed users to the `docker` group. Note: docker group membership is equivalent to root access on the host.

## Secrets management

Sensitive values should be stored in Ansible Vault:

```bash
# Encrypt a value inline
ansible-vault encrypt_string 'my-secret-key' --name 'netbox_secret_key'
```

```yaml
# group_vars/netbox.yml
netbox_secret_key: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  ...
netbox_db_password: !vault |
  ...
netbox_superuser_password: !vault |
  ...
```

## Playbooks

### Deploy / converge

```bash
ansible-playbook playbook.yml
```

Idempotent — safe to run repeatedly. Re-runs do not restart containers unless configuration changed.

### Update NetBox version

```bash
ansible-playbook playbook.yml \
  -e netbox_version=v4.3 \
  -e netbox_update=true
```

Update flow:
1. pg_dump backup (if `netbox_pre_update_backup: true`)
2. Pull new upstream image or rebuild custom image with `nocache: true`
3. Recreate all containers
4. Wait for `/login/` to return 200
5. Run `manage.py migrate`

### Backup

```bash
# Database + media
ansible-playbook backup.yml

# Database only
ansible-playbook backup.yml -e netbox_backup_media=false
```

Backup variables:

| Variable | Default | Description |
|---|---|---|
| `netbox_backup_dir` | `/opt/netbox-backups` | Where to write backups |
| `netbox_backup_media` | `true` | Also archive the media volume |
| `netbox_backup_keep` | `7` | Backups to retain (0 = keep all) |

### Restore

```bash
ansible-playbook restore.yml \
  -e netbox_restore_backup=/opt/netbox-backups/netbox-db-20260614T020000.sql

# With media restore
ansible-playbook restore.yml \
  -e netbox_restore_backup=/opt/netbox-backups/netbox-db-20260614T020000.sql \
  -e netbox_restore_media=/opt/netbox-backups/netbox-media-20260614T020000.tar.gz
```

> **Warning:** restore drops and recreates the NetBox database. Take a backup first.

Restore flow: stops app containers → drops/recreates DB → loads dump → restores media (via temp container, no app container needed) → starts stack → health check → migrate.

## Scheduled backups

Enable a systemd timer on the target host:

```yaml
# group_vars/netbox.yml
netbox_backup_schedule_enabled: true
netbox_backup_on_calendar: "*-*-* 02:00:00"   # daily at 2am (systemd OnCalendar)
netbox_backup_random_delay: "10min"            # jitter across hosts
netbox_backup_keep: 7
```

Then re-run `ansible-playbook playbook.yml`. This deploys:

- `/usr/local/bin/netbox-backup` — standalone bash script (no Ansible at runtime)
- `netbox-backup.service` — oneshot unit, logs to journald
- `netbox-backup.timer` — fires on schedule with `Persistent=true` (catches missed runs after reboots)

Useful commands on the target host:

```bash
systemctl list-timers netbox-backup.timer   # show next scheduled run
systemctl start netbox-backup.service       # run immediately
journalctl -u netbox-backup -f              # stream logs
```

## Docker services

The Compose stack includes:

| Service | Image |
|---|---|
| `netbox` | `netboxcommunity/netbox:<version>` (or custom build) |
| `netbox-worker` | same |
| `netbox-housekeeping` | same |
| `postgres` | `postgres:16-alpine` |
| `redis` | `redis:7-alpine` |
| `redis-cache` | `redis:7-alpine` |

All containers have `restart: unless-stopped` and Docker is enabled on boot, so the stack survives reboots automatically.

## Radarr / custom list integration

NetBox is not related to Radarr — see the sibling `movie-list` project for that.
