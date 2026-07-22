# Diode OAuth2 client registry

Plaintext, non-secret index of every Diode client that exists - what it's
for, what consumes it, and where its actual secret value lives. This file
never contains a secret itself; it only ever records *where* one is
stored. Keep this up to date whenever a client is added or removed - it's
the one place someone unfamiliar with this repo's Ansible structure or
without vault access can look and immediately understand what exists and
why (see `docs/diode-adding-oauth2-clients.md` for why the NetBox GUI
can't be used for this).

**This file can drift from reality - the vault can't lie to itself, but a
doc can get stale.** Always cross-check against the live, authoritative
list before trusting this table for anything consequential:
```bash
docker exec diode-hydra-1 hydra list oauth2-clients --endpoint http://127.0.0.1:4445 --format json \
  | jq -r '.items[] | "\(.client_id)\t\(.client_name)\t\(.scope)\t\(.created_at)"'
```

## Naming convention

`<purpose>-<location>` - e.g. `orb-agent-vhohetzner1`. Established
2026-07-22 specifically because the original three clients
(`diode-to-netbox`, `netbox-to-diode`, `orb-agent`) only worked as names
because there was exactly one of each *kind* - that breaks the moment a
second Orb Agent (or sync job) exists anywhere else. Every new client
going forward should follow this pattern; the original three are grandfathered in and not worth renaming (renaming means recreating, and they're
already unambiguous as the only host in play today).

## Registry

| Client ID | Purpose | Consumer | Scope | Secret location | Added |
|---|---|---|---|---|---|
| `diode-to-netbox` | Diode reconciler -> NetBox plugin API | Diode server (vhohetzner1) | `netbox:read netbox:write` | vault: `diode_to_netbox_client_secret` (`group_vars/Hetzner/vault.yml`) | pre-2026-07-21 |
| `netbox-to-diode` | NetBox plugin -> Diode | NetBox Diode plugin (vhohetzner1) | `diode:read diode:write` | vault: `netbox_to_diode_client_secret` (`group_vars/Hetzner/vault.yml`) | pre-2026-07-21 |
| `orb-agent` | Discovery ingestion | Orb Agent (vhohetzner1) - `network_discovery` + `snmp_discovery` | `diode:ingest` | vault: `orb_diode_client_secret` (`group_vars/Hetzner/vault.yml`) | pre-2026-07-21 |

Rename this table's `orb-agent` row to `orb-agent-vhohetzner1` the next
time that client is ever recreated for any reason - not urgent enough to
force a recreation just for the rename today.

## Test/throwaway clients created during investigation (2026-07-22)

Not part of the working system - listed here only so nobody mistakes them
for something load-bearing if they're still lingering in Hydra:

| Client ID | Purpose | Status |
|---|---|---|
| `verify-test-001` | Verified whether `authmanager`-created clients show on the GUI (they don't) | Deleted after test |
| `test-manual-cc-f2e838ad83c94781` | Verified the GUI's "Add a Credential" flow and its `owner` tagging | Still live as of 2026-07-22 - delete via the GUI or `hydra delete oauth2-client` once no longer needed, it authenticates fine and does nothing harmful sitting idle |
| `3e2b015c-7ed0-4b21-8e40-0b9a63bfe144` | Verified (and disproved) that `hydra update oauth2-client --owner` is a safe partial update | **Broken on purpose** - confirmed dead per `docs/diode-adding-oauth2-clients.md`'s DANGER section. Delete it; don't reuse the ID. |

## When onboarding a new site (see `docs/multi-site-diode-architecture.md`)

Each new site's Orb Agent gets its own row here the moment it's
provisioned - `orb-agent-<site>`, scope `diode:ingest`, secret location
pointing at wherever that site's vault entry lives. Same for any future
central-sync-job credentials (`sync-<site>-to-central`) once that's built.
