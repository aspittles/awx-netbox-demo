# AWX + NetBox Automation Webinar Demo

This repo powers a three-part demo showing NetBox as a source of truth for
infrastructure automation, with AWX as the execution engine and Event-Driven
Ansible (EDA) as the trigger mechanism.

## Demos

1. **Dynamic Inventory** — AWX pulls a live inventory from NetBox, filtered
   down to a specific site and device type (Acme Sydney, Dell PowerEdge R750).
2. **Event-Driven Automation** — A NetBox webhook fires when a tag is added to
   a VM; an `ansible-rulebook` listener reacts and automatically launches an
   AWX Job Template — no manual trigger.
3. **Config Deployment** — NetBox's Config Context + Config Template render a
   real configuration via the `render-config` API; an EDA-triggered AWX job
   deploys it additively to a live `chrony` NTP setup.

## Files

### Root

| File | Used by | Purpose |
|---|---|---|
| `netbox_inv_sydney_r750.yml` | Demo 1 | `netbox.netbox.nb_inventory` plugin config, filtered to `site: acme-apj-sydney` + `device_type: dell-poweredge-r750` |
| `netbox_inv.yml` | (reference only) | Original unfiltered NetBox inventory — full device list, grouped by role/site. Not part of the live webinar flow, kept as a working reference/fallback. |
| `netbox_inv_eda_target.yml` | Demo 2 & 3 | Dynamic inventory filtered to the `eda-demo` tag — resolves to `NetBoxWebinar01` itself. Exposes `ansible_host` (IP, CIDR stripped) and `netbox_vm_id` (NetBox's internal VM ID) as host vars. |

### `collections/`

| File | Purpose |
|---|---|
| `requirements.yml` | Pins the `netbox.netbox` collection (`>=3.20.0`) that AWX installs automatically on every Project sync — required for the `nb_inventory` plugin used by all three inventory files above. |

### `playbooks/`

| File | Used by | Purpose |
|---|---|---|
| `eda-welcome-banner.yml` | Demo 2 | Runs against `NetBoxWebinar01`. Ensures `/etc/motd` exists, adds a one-time header (idempotent via `blockinfile`), then appends a timestamped line on every run (via `lineinfile`) plus drops a marker file in `/tmp`. |
| `netbox-config-deploy.yml` | Demo 3 | Calls NetBox's `render-config` API for `NetBoxWebinar01` (VM ID resolved dynamically via `netbox_vm_id` from inventory, not hardcoded), writes the rendered output to `/etc/chrony/conf.d/netbox-demo.conf`, and restarts `chrony`. Requires the `NETBOX_TOKEN` env var (write-scoped token) injected via credential. |

### `eda/`

| File | Purpose |
|---|---|
| `webinar-rulebook.yml` | The Event-Driven Ansible rulebook. Listens on port 5050 via `ansible.eda.webhook`. Contains two rules: one matches the `eda-notify-demo` tag being newly added (triggers `EDA Demo - Welcome Banner`), the other matches `config-deploy-demo` being newly added (triggers `NetBox Config Deploy`). Both conditions check the webhook's `snapshots.prechange`/`data.tags` to detect a genuine tag **transition**, not just presence — this avoids both rules firing simultaneously if the VM happens to carry both tags at once. |
| `inventory.yml` | Placeholder inventory required by the `ansible-rulebook` CLI itself (`-i` flag) — not related to the real AWX inventories; the rulebook's actions call AWX's API directly rather than running anything locally. |

## AWX Setup Summary

- **Project:** `NetBox Dynamic Inventory Demo` — this GitHub repo, synced on demand (and whenever new files are pushed, sync the Project *before* syncing any Inventory Source that depends on it).
- **Credentials:**
  - `NetBox Cloud Demo` — read-only NetBox API token, used by all three inventory sources.
  - `NetBox Cloud Demo - Write` — read/write NetBox API token, used only by the `NetBox Config Deploy` Job Template (needed for the `render-config` POST endpoint).
  - `NetBoxWebinar01 SSH` — Machine credential (SSH + sudo) for the two Job Templates that connect to `NetBoxWebinar01` directly.
- **Inventories:**
  - `NetBox – Acme Sydney R750s` → sourced from `netbox_inv_sydney_r750.yml`
  - `NetBox Devices` → sourced from `netbox_inv.yml` (kept as reference/fallback, not used live)
  - `EDA Target (NetBox)` → sourced from `netbox_inv_eda_target.yml`
- **Job Templates:**
  - `EDA Demo - Welcome Banner` — Demo 2, uses `EDA Target (NetBox)` + `playbooks/eda-welcome-banner.yml` + `NetBoxWebinar01 SSH`
  - `NetBox Config Deploy` — Demo 3, uses `EDA Target (NetBox)` + `playbooks/netbox-config-deploy.yml` + both `NetBoxWebinar01 SSH` and `NetBox Cloud Demo - Write`

## NetBox Setup Summary

- **Tags:** `eda-demo` (permanent, scopes the EDA target inventory), `eda-notify-demo` and `config-deploy-demo` (ephemeral — added live during the webinar to fire each demo's webhook, safe to add/remove without side effects)
- **Config Context:** `Webinar Demo NTP Config`, scoped by **Site** (`adrian-home`) rather than tag — contains `ntp_servers` and `banner_text`
- **Config Template:** `Chrony NTP Config`, assigned to the `NetBoxWebinar01` VM — renders the Config Context data into real chrony `server ... iburst` directives
- **Webhook:** `EDA Rulebook Trigger` → POSTs to `http://<public-facing-address>:5050/endpoint` (NAT-forwarded to `NetBoxWebinar01`)
- **Event Rule:** `EDA Demo Trigger Rule` → fires on Virtual Machine Updates, calls the webhook above

## Reset Between Rehearsals

A local script (`~/Webinar/reset-demo.sh` on `NetBoxWebinar01`, not checked into this repo) clears `/etc/motd`, removes the `/tmp` marker files, and removes/restarts the chrony demo config — restoring both Demo 2 and Demo 3 to a clean pre-demo state. Remember to also remove the `eda-notify-demo` / `config-deploy-demo` tags in NetBox, and clear the hosts from `EDA Target (NetBox)` / `NetBox – Acme Sydney R750s` in AWX if you want the "empty inventory populates live" opening beat for Demo 1.
