# Agent instructions - ansible-vholab

This is a personal homelab Ansible repo (see `README.md` for the full
playbook/application map). Several of its bugs have taken long, multi-step
live-debugging sessions to actually root-cause - the kind of thing that's
cheap to check and expensive to re-derive from scratch. `docs/` also holds
forward-looking design-decision records for work that hasn't been built
yet - read those before assuming a piece of architecture is undecided.

## Before debugging an issue, or planning new work, in this repo

**Check `docs/` first**, before reasoning from scratch. Most entries
document one hard-won incident: the actual root cause, the dead ends that
looked plausible but weren't (worth reading even if the specific symptom
doesn't match - the *reasoning traps* often recur across unrelated
services), and the real fix - if a live symptom resembles one of these,
don't re-walk the same dead ends, jump straight to verifying whether the
same root cause applies here too. A few entries instead record a design
decision for planned-but-not-yet-built work, including the options that
were considered and rejected - read these before re-opening a design
question that's already been settled.

Current entries:
- `docs/snmp-discovery-persistent-path-bug.md` - SNMPv3 discovery silently
  never worked on either Hetzner host because the role wrote its
  `createUser` line to `/var/lib/snmpd/` instead of net-snmp's real
  persistent path, `/var/lib/snmp/` (no trailing "d"). Also documents a
  reusable technique (foreground + full debug trace, captured to a file)
  for when a daemon's live behavior doesn't match its config and
  `journalctl` isn't explaining why.
- `docs/netbox-diode-secret-permission-bug.md` - NetBox's Diode plugin
  reported a missing client secret even though the file existed on the host
  with real content, because standalone `docker compose secrets:` bind-mounts
  the host file as-is (no re-chown for the container's user namespace), and
  the secret was `root:root 0600` while the container runs as a non-root
  user. Same bug class as an already-fixed one elsewhere in this repo
  (Diode's own `client-credentials.json`). Fixed and confirmed live. Quick
  reusable check: `docker compose exec <svc> wc -c <path>` as the
  container's default user - "exists on the host" and "readable inside the
  container" are different questions.
- `docs/snmp-discovery-does-not-run-on-restart.md` - corrects a `README.md`
  claim: restarting Orb Agent re-runs `network_discovery` immediately (true,
  has its own `"running scanner"` log line), but does **not** trigger an
  early `snmp_discovery` probe - that backend only actually scans on its own
  `orb_snmp_schedule` cron tick, confirmed by grepping a full day of logs
  for the backend-specific `"starting SNMP probe scan"` line vs. the
  always-present-but-meaningless `"policy applied successfully"`.
- `docs/diode-adding-oauth2-clients.md` - the NetBox Diode plugin's Client
  Credentials GUI page filters on Hydra's own `owner` field, literally
  `"diode/user"`, set only by the GUI's own "Add a Credential" button (not
  by this repo's `bootstrap-clients.sh` or NetBox Labs' `authmanager` CLI -
  both talk to Hydra directly and never set it). **Important safety
  finding**: `hydra update oauth2-client --owner ...`, despite Ory's own
  docs claiming it only changes explicitly-passed fields, was tested live
  and confirmed to silently reset grant_types/scope/token-auth-method to
  defaults on this pinned version (`v26.2.0`) and break a working client's
  secret in the process - **never run this against `diode-to-netbox`,
  `netbox-to-diode`, or `orb-agent`** to try to retrofit GUI visibility.
  There is no safe way to add the tag to an existing client; only creating
  a new one through the GUI from the start works.
- `docs/diode-clients-registry.md` - plaintext (non-secret) index of every
  Diode OAuth2 client: what it's for, what consumes it, where its actual
  secret lives (vault key, not the value). Established alongside a
  naming convention (`<purpose>-<location>`, e.g. `orb-agent-vhohetzner1`)
  since the original three clients' names only worked because there was
  exactly one of each kind - keep both updated whenever a client is
  added/removed, and cross-check against `hydra list oauth2-clients`
  (the always-authoritative source) before trusting this file for
  anything consequential.
- `docs/multi-site-diode-architecture.md` - design-decision record (not
  yet built) for extending this repo to multiple sites, each with its own
  autonomous NetBox+Diode+Orb Agent, plus a central NetBox for cross-site
  visibility. Documents why "central NetBox+Diode, remote Orb-Agent-only"
  doesn't satisfy the actual requirement (site teams need real local
  ownership), why NetBox Enterprise's multi-tenant platform is out of
  scope (customer is on Community), and the chosen approach: a custom
  sync script per site using `netboxlabs-diode-sdk` to push each site's
  NetBox inventory into the central Diode pipeline as an ingestion source,
  reusing existing dedup/reconciliation machinery instead of building a
  bespoke NetBox-to-NetBox sync system. Read this before re-litigating the
  central-visibility design from scratch.

## When you resolve a similarly hard-won bug

Add a new file to `docs/` documenting it, in the same shape as the existing
one: symptom, root cause, dead ends ruled out (and why they looked
plausible), the fix, and how to verify it. Skip this for anything
straightforward (a typo, a missing var, a one-line config fix with an
obvious cause) - this is for issues that cost real back-and-forth to
actually diagnose, where a future session (yours or someone else's) would
otherwise burn the same time again.
