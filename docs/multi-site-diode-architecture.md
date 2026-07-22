# Multi-site NetBox + Diode architecture (Community edition)

Status: **planned, not yet implemented.** This is a design-decision record,
not a bug writeup - captures the constraints found and the option chosen
before any of it is built, so the reasoning survives even if
implementation happens much later or by someone else.

## Context / requirement

Multiple sites, each with its own team that manages its own network
infrastructure. Requirements, as stated:

- Each site needs its **own, fully autonomous NetBox instance** - the site
  team owns and edits it directly. Not a remote agent feeding someone
  else's system; a real local instance they control.
- A **central NetBox** needs visibility across all sites.
- Customer is on **NetBox Community**, not Enterprise - the Enterprise
  multi-tenant Diode/NetBox platform this same machine's `upcloud-infra`
  repo is piloting is explicitly not available here. Any solution has to
  work with the self-hosted community stack this repo already builds
  (`roles/netbox`, `roles/diode_server`, `roles/orb_agent`,
  `roles/netbox_diode_plugin`).

## Constraints found while designing this (the "issues")

1. **`netbox_diode_plugin` (community) is a strict 1:1 pairing.** NetBox's
   Diode plugin is configured to trust tokens from exactly one Diode
   server/Hydra instance at a time (see
   `docs/diode-adding-oauth2-clients.md` for how that trust/ownership
   mechanism actually works). There is no way for one central NetBox to
   accept ingestion directly from N independent remote Diode servers - it
   isn't a multi-backend-aware plugin.
2. **NetBox has no built-in federation.** Nothing in NetBox itself merges
   or aggregates data across multiple independent instances. Any
   cross-site view has to be built, not configured.
3. Consequence of (1): a "central NetBox + Diode, remote sites run only
   Orb Agent" model (the simplest fit for how Diode is actually designed -
   see the discussion this doc's history is based on) does **not** satisfy
   the actual requirement, since site teams need real local ownership of
   their own NetBox, not just an agent reporting into a system they don't
   control.

## The plan

- Each site gets a **full, independent stack** via this same Ansible
  repo - local NetBox + local Diode server + local Orb Agent, exactly the
  roles already built and proven working end-to-end on vhohetzner1/
  vhohetzner2 this session. Onboarding a new site means adding it to
  inventory/host_vars and running the existing `netbox_deploy.yml` +
  `netbox_discovery_deploy.yml`, unchanged.
- Each site's team owns and operates their own NetBox instance fully - it
  is the real source of truth for that site's inventory. Nothing about
  this plan changes that.
- A separate **central NetBox** exists purely for cross-site visibility -
  it is explicitly NOT the authoritative source for any site's data, and
  should never be treated as one.

## Options considered for the central-visibility layer

1. **Bespoke NetBox-API-to-NetBox-API sync script** (e.g. `pynetbox`,
   plain REST calls) pulling from each site and writing into central.
   Straightforward, but reinvents dedup/idempotency/conflict-handling from
   scratch - a second bespoke system to build and maintain, parallel to
   the Diode machinery this repo already has working.
2. **Read-only aggregation dashboard**, central queries each site's API
   live (or short-cached) and never stores a copy. Avoids stale-data /
   two-sources-of-truth problems entirely, but doesn't give a queryable,
   NetBox-native central inventory - harder to do things like cross-site
   IPAM conflict checks or unified reporting the way a real NetBox
   instance supports natively.
3. **NetBox Enterprise's actual multi-tenant platform.** The properly
   supported answer to this exact problem shape - see `upcloud-infra` on
   this machine. Ruled out here: customer is explicitly on the Community
   tier, not evaluating Enterprise.
4. **Reuse the central Diode pipeline as the sync transport (chosen).** A
   small custom script, using the documented `netboxlabs-diode-sdk` Python
   SDK, pulls objects from each site's NetBox REST API (read-only token per
   site) and pushes them as Diode ingestion entities into **one** central
   Diode+NetBox pairing - the same kind of pairing already proven working
   this session, just with "read a remote NetBox's API" as the discovery
   backend instead of SNMP/nmap.

## Why option 4

- **Reuses proven machinery** instead of building a parallel system:
  dedup, reconciliation, the changeset-review workflow, and audit trail
  (the `diode` changelog username, same as everywhere else in this repo)
  all come for free, rather than re-implementing idempotency/conflict
  handling by hand.
- **Fits the naming-convention + registry already established**
  (`docs/diode-clients-registry.md`) - each site's sync job gets its own
  uniquely-named, ingest-scoped credential (e.g. `sync-siteB-to-central`),
  the same pattern as `orb-agent-siteB`.
- **Respects the 1:1 plugin constraint** rather than fighting it - the
  central NetBox's Diode plugin still only ever trusts its one paired
  Diode/Hydra instance; this sync script is just another ingestion source
  feeding *into* that same central Diode, not a second Diode backend
  trying to talk to NetBox directly.
- **No special trust relationship needed on the site side** - central
  never talks to a site's NetBox except via a plain, scoped, read-only
  REST API token. Sites don't need to open anything up beyond that.

## Design decisions still open (not yet implemented)

- Which objects/fields actually need central visibility - likely
  Sites/Devices/Interfaces/Prefixes/IP Addresses; probably not
  Circuits/VPN/etc. unless a concrete need shows up.
- Tagging convention for synced objects so central never treats them as
  locally editable - e.g. a custom field or tag like `source: <site>-sync`
  on everything the sync script creates/updates.
- Sync cadence and incremental-pull mechanism - NetBox's
  `/api/extras/object-changes/` changelog endpoint, tracking a
  last-synced timestamp per site, rather than re-pulling everything each
  run.
- Whether `autoApplyChangesets` should be off for this specific ingestion
  source (human review of cross-site merges before they land centrally),
  even if local Orb Agent discovery at each site stays auto-apply.
- Inventory/host_vars structure for onboarding a new site - a new
  Ansible group per site, or shared role defaults with per-site overrides?
- Where the sync script itself runs (cron on the central host vs. a small
  container) and how its own credentials (per-site read tokens + the
  central ingest credential) get vaulted and rotated - same discipline as
  everything else in `group_vars/Hetzner/vault.yml`.

## Next step

Pick a first pilot site to onboard under this pattern, and design the
actual sync script + inventory structure against a real site rather than
in the abstract.
