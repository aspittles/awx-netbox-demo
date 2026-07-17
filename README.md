# AWX NetBox Dynamic Inventory Demo

Demo project showing AWX pulling a dynamic inventory from NetBox using the
`netbox.netbox.nb_inventory` plugin.

## Files
- `netbox_inv.yml` — inventory plugin config (grouped by device role and site)
- `collections/requirements.yml` — pins the `netbox.netbox` collection AWX needs to install

## Setup
NetBox API URL and token are supplied via an AWX custom credential
(`NETBOX_API` / `NETBOX_TOKEN` env vars) — not stored in this repo.
