# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Ansible roles for deploying and operating NetBox, importing device types from the community library, and syncing server inventory into NetBox from live hosts.

## Running playbooks

Install the Ansible venv and collections first (one-time):

```bash
python3 -m venv /tmp/ansible-venv && source /tmp/ansible-venv/bin/activate
pip install ansible pynetbox
ansible-galaxy collection install -r requirements.yml -p collections/
```

**Deploy / converge NetBox:**
```bash
ansible-playbook playbook.yml
```

**Import device types from the community library:**
```bash
ansible-playbook import_device_types.yml \
  -e dtl_netbox_url=http://netbox:8080 \
  -e dtl_netbox_token=<token> \
  -e '{"dtl_manufacturers": ["Dell", "Cisco"]}'
```

**Inventory Linux servers into NetBox:**
```bash
ansible-playbook linux_inventory.yml \
  -e linux_inventory_netbox_url=http://netbox:8080 \
  -e linux_inventory_netbox_token=<token> \
  -e linux_inventory_site=<site-slug>
```

**Inventory Windows servers into NetBox:**
```bash
ansible-playbook windows_inventory.yml \
  -e windows_inventory_netbox_url=http://netbox:8080 \
  -e windows_inventory_netbox_token=<token> \
  -e windows_inventory_site=<site-slug>
```

## NetBox API tokens (NetBox 4.5+)

NetBox 4.5 introduced v2 tokens (HMAC-signed). The `netbox.netbox` collection requires v1 (plaintext) tokens. Create one explicitly:

```bash
docker exec <netbox-container> python3 /opt/netbox/netbox/manage.py shell -c "
from users.models import User, Token
u = User.objects.filter(is_superuser=True).first()
t = Token(user=u, write_enabled=True, version=1)
t.save()
print(t.plaintext)
"
```

A v2 token will return `{"detail":"Invalid v1 token"}` from the API.

## Role architecture

### `netbox_docker`
Deploys NetBox via Docker Compose. Manages the full stack (postgres, redis, redis-cache, netbox, netbox-worker, netbox-housekeeping). Builds a custom image when `netbox_plugins` is non-empty. Key vars: `netbox_version`, `netbox_secret_key`, `netbox_db_password`, `netbox_superuser_password` (store last three in vault).

### `device_type_importer`
Clones the [netbox-community/devicetype-library](https://github.com/netbox-community/devicetype-library) (`master` branch, not `main`) and imports device type YAML files into NetBox. Uses `pynetbox` for the import logic. Module bay templates have no `netbox.netbox` module — they are created via `uri` POST to `/api/dcim/module-bay-templates/` with idempotent 400 handling.

### `linux_inventory` / `windows_inventory`

Both roles gather facts from target hosts and sync them into NetBox. The flow:

1. **Normalize facts** → set `_li_*` prefixed vars (hostname, is_vm, manufacturer, model, platform, interfaces list)
2. **Ensure NetBox objects** exist (platform, manufacturer, device type, device role) — all `delegate_to: localhost`
3. **Sync device or VM** (`sync_device.yml` / `sync_vm.yml`) — physical hosts → `netbox_device`, VMs → `netbox_virtual_machine`
4. **Sync interfaces** (`sync_interfaces.yml`) — one iteration per interface; creates interface then assigns IPv4 via `assigned_object: {name: iface, device: hostname}`

`windows_inventory` symlinks its `sync_device.yml`, `sync_vm.yml`, and `sync_interfaces.yml` from `linux_inventory/tasks/` — NetBox sync logic lives in one place. The `windows_inventory` `main.yml` bridges `windows_inventory_*` vars to `linux_inventory_*` names so the shared tasks work unchanged.

**Internal `_li_*` vars** (set in `main.yml`, consumed by sync tasks):

| Var | Source |
|---|---|
| `_li_hostname` | `ansible_hostname` |
| `_li_is_vm` | `ansible_virtualization_role == 'guest'` |
| `_li_platform_name` / `_li_platform_slug` | distribution + version (Linux) / stripped `ansible_os_name` (Windows) |
| `_li_manufacturer` / `_li_manufacturer_slug` | `ansible_system_vendor` (vendor suffixes stripped) |
| `_li_model` / `_li_model_slug` | `ansible_product_name` |
| `_li_serial` | `ansible_product_serial` (sanitised; empty if placeholder value) |
| `_li_interfaces` | `ansible_interfaces` filtered by `*_skip_iface_re` |

**IP address assignment** uses `assigned_object: {name: "<iface>", device: "<hostname>"}` (string values, not dicts) — this is the correct format for `netbox.netbox` ≥ 3.19.

## Key conventions

- **Secrets in vault**: `netbox_secret_key`, `netbox_db_password`, `netbox_superuser_password`, and Windows `ansible_password` must never be in plaintext.
- **`device_role`** not `role` in `netbox_device` data — the collection uses the old field name.
- **Platform references** by `slug:`, not `name:` — the `netbox_device` and `netbox_virtual_machine` modules reject `name:` for platform.
- **`linux_inventory` runs `become: false`** — facts don't need root. MAC addresses may be absent on some hosts without privilege escalation.
- **Windows connection**: WinRM defaults (`ntlm`, port 5985) are set in the `windows_servers` inventory group. Add `ansible_password` per host in vault.
- **`dtl_library_version`** must be `master` (the community devicetype-library default branch, not `main`).
