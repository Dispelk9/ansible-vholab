# NetBox Diode plugin: "Missing netbox to diode client secret"

Status: **fixed** (`roles/netbox_diode_plugin/tasks/secrets.yml`), 2026-07-21.

## Symptom

NetBox's Diode plugin **Client Credentials** page shows a persistent red
banner: *"Please update the plugin configuration to access this feature.
Missing netbox to diode client secret."* - even though the secret file was
confirmed present on the host with real content (`wc -c
.../secrets/netbox_to_diode` showed 44 bytes, not empty/missing), and the
`netbox`/`netbox-worker` containers were confirmed running the correct
custom Diode-plugin image (`netbox-diode-custom:diode-1.12.0`, not the
plain upstream one).

(Separately: this page shows **0 results** even when everything is healthy
- that part is expected, unrelated, and documented in the main
`README.md` under "Client Credentials page won't show Orb's (or Diode's
own) client". Don't confuse the two - an empty results table is normal;
the red banner is the actual bug.)

## Root cause

Per `roles/netbox_diode_plugin/templates/plugins_block.py.j2`, the plugin
deliberately does **not** get its client secret from
`configuration/plugins.py` - it reads it at runtime from a Docker secret
mounted at `/run/secrets/netbox_to_diode`, sourced from
`{{ netbox_diode_build_dir }}/secrets/netbox_to_diode` on the host (wired
up via `docker-compose.diode.yml.j2`'s `secrets:` block).

`tasks/secrets.yml` wrote that file (and its parent directory) as
`root:root`, mode `0700`/`0600`. Standalone `docker compose` (not Swarm)
implements `secrets:` as a **plain bind-mount of the host file, as-is** -
it does not re-chown or re-permission anything for the container's user
namespace. The `netbox`/`netbox-worker` containers run as a non-root user
(confirmed live: `docker compose exec netbox wc -c
/run/secrets/netbox_to_diode` using the container's default exec user
returned `Permission denied`), so that user could never read a
`root:root 0600` file, no matter how many times the containers were
recreated.

This is the *exact same bug class* already fixed once in this repo for
Diode's own OAuth2 `client-credentials.json`
(`roles/diode_server/templates/client-credentials.json.j2`, deliberately
`0644`/dir `0755` - see the comment there, and `netbox_manual_deploy.md`) -
it just hadn't been applied to this second, separate secret file yet.

## The fix

`roles/netbox_diode_plugin/tasks/secrets.yml`: directory `0755`, file
`0644` (both still `root`-owned - only the read bit needed loosening, not
ownership). Matches the established, working precedent for
`client-credentials.json` instead of inventing a new pattern.

## Why this one was easy (contrast with the SNMP bug)

Unlike `docs/snmp-discovery-persistent-path-bug.md`, this bug did **not**
need a live debug trace to diagnose - the smoking gun was one command:
```bash
docker compose exec netbox wc -c /run/secrets/netbox_to_diode
# Permission denied
```
A single "can the container's own user actually read this file" check
settled it immediately. Worth trying *before* reaching for anything
heavier whenever a containerized process claims a mounted
file/secret/config is missing but the host confirms it exists with real
content - "exists on the host" and "readable by the process inside the
container" are different questions, especially with non-Swarm `docker
compose secrets:`, which (unlike real Docker/Swarm secrets) never adjusts
permissions for you.

## Verifying this is fixed

```bash
ansible-playbook -i inventory netbox_discovery_deploy.yml --tags netbox_diode_plugin
ls -la /opt/netbox/discovery/netbox-diode-build/secrets/netbox_to_diode   # should show -rw-r--r--
docker compose exec netbox wc -c /run/secrets/netbox_to_diode            # should print the byte count, not "Permission denied"
```
Then reload NetBox's Diode plugin **Client Credentials** page - the red
banner should be gone (the "0 results" table staying empty is still
normal, see above).
