# Adding another Diode OAuth2 client

Status: **confirmed live**, 2026-07-22 - tested all three creation paths
(the existing file-based bootstrap, `authmanager`, and the GUI's own "Add a
Credential" button) and compared their actual Hydra records directly,
rather than inferring from docs. This doc went through two wrong guesses
before landing here - see the note at the bottom if you want the history.

## The real, confirmed mechanism

NetBox's Diode plugin **Client Credentials** page (Diode menu -> Client
Credentials) filters on Hydra's own `owner` field, and it's a literal,
specific string: `"diode/user"`. Confirmed by creating one throwaway client
each way and inspecting `hydra list oauth2-clients` directly:

| Creation method | Hydra `owner` field | Shows on GUI page |
|---|---|---|
| `roles/diode_server/templates/client-credentials.json.j2` + `bootstrap-clients.sh` (this repo's existing 3: `diode-to-netbox`, `netbox-to-diode`, `orb-agent`) | `""` | No |
| `authmanager create-client` | `""` | No |
| GUI's own **"Add a Credential"** button | `"diode/user"` | **Yes** |

**Only the GUI's own creation flow calls `diode-auth`'s REST API**, which
is what stamps `owner: "diode/user"` onto the Hydra client it creates.
Both `bootstrap-clients.sh` *and* `authmanager` - despite being two
different tools - both talk to Hydra's admin API **directly**, bypassing
`diode-auth`'s REST layer (and therefore the owner-tagging) entirely. This
is a deliberate design split, not an inconsistency: the GUI path is for
one-off, interactively-created credentials an admin wants tracked in
NetBox's own UI; `authmanager`/the bootstrap file are for infrastructure-
level service clients (Diode's own internal wiring, persistent agents)
that don't need day-to-day UI visibility.

## DANGER: do not try to retrofit `owner` onto an existing client

It's tempting, having read the table above, to think "just patch the
existing 3 clients' `owner` field and they'll show up too, no need to
recreate anything." **Tested live and confirmed destructive - do not do
this:**

```bash
docker exec diode-hydra-1 hydra update oauth2-client <id> \
  --owner "diode/user" --endpoint http://127.0.0.1:4445
```

Ory's own CLI docs claim "only the fields provided as flags are changed;
all other fields keep their current values." **This is false for
`v26.2.0` (the version pinned in this repo).** Tested on a disposable
client: before the update it had `grant_types: client_credentials`,
`scope: diode:ingest`, and a known working secret (confirmed via `hydra
perform client-credentials` returning a real access token). Running
*only* `--owner "diode/user"` against it reset **grant_types to
`authorization_code`, scope to `offline_access offline openid`, and the
token auth method** - all silently defaulted, none of them touched by the
command. The identical secret that worked seconds earlier then failed with
`invalid_client` on the exact same `hydra perform client-credentials`
call. `hydra update oauth2-client` is a full replace/reset for any field
you don't explicitly pass, not a merge - despite what its own docs say.

Running this against `diode-to-netbox`, `netbox-to-diode`, or `orb-agent`
would have broken the entire working Diode pipeline this repo depends on.
**There is no safe way to add the GUI-visibility tag to an already-existing
client.** The only path to a GUI-visible client is creating a *new* one
through the GUI's own "Add a Credential" button from the start (see below)
- never by modifying one that already exists and is in use.

## The GUI's "Add a Credential" form

Three plain fields: **Client Name**, **ID**, **Secret** - ID and Secret are
both auto-generated for you (no need to run your own `openssl rand` or
similar); Client Name is your own label. Submitting it creates a real,
usable Hydra OAuth2 client (confirmed - it's fully present and functional
in `hydra list oauth2-clients`, not just a local NetBox-side bookkeeping
record), with `owner: "diode/user"` set automatically, `scope:
"diode:ingest"` by default.

Don't bother vaulting a credential created this way unless it's genuinely
meant to be permanent - `group_vars/Hetzner/vault.yml` is for secrets this
repo's Ansible roles actually template into config files; a GUI-created
credential isn't consumed by anything in this repo, so vaulting it here
would just be an orphaned entry nothing reads. A password manager is a more
honest home for it if you want to keep it around.

## The officially documented CLI way: `authmanager`

Confirmed live in NetBox Labs' own `netboxlabs/diode` repo
(`diode-server/README.md`, "Provisioning and Managing Agent Credentials via
Command Line") - a real, first-party CLI bundled in the `diode-auth` image:

```bash
cd /opt/netbox/discovery/diode

# Create a new client for ingestion (auto-generated secret, shown once - save it now)
docker compose run --rm --no-deps diode-auth authmanager create-client --client-id my-agent-001 --allow-ingest

# Or supply your own secret instead of an auto-generated one
docker compose run --rm --no-deps diode-auth authmanager create-client \
  --client-id my-agent-002 --allow-ingest --client-secret="<your-secret>"

# Inspect / list / remove
docker compose run --rm --no-deps diode-auth authmanager get-client --client-id my-agent-001
docker compose run --rm --no-deps diode-auth authmanager list-clients
docker compose run --rm --no-deps diode-auth authmanager delete-client --client-id my-agent-002
```

Recommended for infrastructure-style clients (another discovery agent, a
script that only needs `diode:ingest`) - simpler than hand-editing JSON and
re-running a role. It **will not** show up on the Client Credentials GUI
page (see table above) - that's expected, use `hydra list oauth2-clients`
to verify it instead.

## The existing (file-based) method - for consistency with the current 3, not recommended for new ones

If you specifically need a new client provisioned the *same way* as the
existing three (e.g. version-controlled in this repo rather than a
one-off imperative command):

1. Add a new entry to `roles/diode_server/templates/client-credentials.json.j2`'s array.
2. Add the corresponding secret to `group_vars/Hetzner/vault.yml`.
3. Re-run just that role: `ansible-playbook -i inventory netbox_discovery_deploy.yml --tags diode_server`
   - this re-templates the file and does `docker compose up -d`, re-running
     the one-shot `diode-auth-bootstrap` job against the *existing* Hydra
     DB. Normal redeploys preserve Diode's volumes
     (`diode_server_reset_on_deploy` defaults `false`, see `README.md`), so
     this is safe - no reset needed.
4. Same GUI-invisibility as the existing three (`owner:""`) - expected,
   and **permanent** - see the DANGER section above before considering
   "fixing" this after the fact.

## Verifying any client, regardless of method

The GUI can only ever show clients created through itself - always check
Hydra directly for the full picture:
```bash
docker exec diode-hydra-1 hydra list oauth2-clients --endpoint http://127.0.0.1:4445 --format json
```

## Choosing which method to use

- **Need it visible/manageable in the NetBox UI** (a human will look it up
  there later) -> use the GUI's "Add a Credential" button.
- **Infrastructure/service client, managed as code or via CLI, UI
  visibility doesn't matter** -> `authmanager` (simplest) or the file-based
  method (if it needs to be version-controlled alongside the existing 3).

<details>
<summary>History: two wrong guesses before this was confirmed</summary>

1. First guess: assumed `authmanager`, being "diode-auth's own bundled
   CLI," would go through the same code path as the GUI and produce
   GUI-visible clients. Tested and disproven - it produced `owner:""`,
   identical to the file-based method.
2. Second guess: pivoted to blaming NetBox-side ownership tied to the
   `diode_username` plugin config setting (`PLUGINS_CONFIG[...]
   ["diode_username"]`, via `DefaultTokenOwnershipProvider`/`get_diode_user()`).
   This conflated two unrelated mechanisms - `diode_username` governs who
   NetBox's own *changelog* attributes applied changesets to (the `diode`
   username visible in NetBox's change log for reconciled objects), not
   Hydra's OAuth2 client `owner` field at all. Disproven by actually
   creating a client through the GUI and inspecting Hydra directly: the
   real answer was `owner: "diode/user"`, a fixed literal string set by
   `diode-auth`'s own REST layer - nothing to do with `diode_username`.

Lesson: for questions like "does X show up in Y," test it directly with a
throwaway object and inspect the actual record, rather than reasoning
from a plausible-sounding mechanism found in secondary docs (DeepWiki
summaries, in this case) that don't fully explain themselves.

</details>
