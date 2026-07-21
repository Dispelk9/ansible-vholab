# ansible-vholab

Ansible configuration for the vho lab: a mix of Hetzner cloud VPS, home-lab
KVM hosts, and a handful of applications (NetBox/Diode/Orb Agent, OTRS/Znuny,
BlueCat) running on top of them. Each application below is a **separate,
independent playbook** - there is no single "deploy everything" entry point,
run only the one you need.

## Table of contents

- [Prerequisites](#prerequisites)
- [Playbook -> application quick reference](#playbook---application-quick-reference)
- [Inventory groups at a glance](#inventory-groups-at-a-glance)
- [Applications](#applications)
  - [NetBox](#netbox--netbox_deployyml)
  - [Diode + Orb Agent discovery (around NetBox)](#diode--orb-agent-discovery-around-netbox--netbox_discovery_deployyml)
  - [SNMP discovery agents on Hetzner hosts](#snmp-discovery-agents-on-hetzner-hosts--update_hetzneryml)
  - [OTRS / Znuny](#otrs--znuny--play-otrsyml)
  - [BlueCat Lab](#bluecat-lab--bluecat_deployyml)
  - [KVM provisioning](#kvm-provisioning--kvm_initialyml)
  - [Lab hosts bootstrap](#lab-hosts-bootstrap--hostall_inityml)
  - [Flask app (ACT) on vhohetzner1](#flask-app-act-on-vhohetzner1--vhoflask_inityml)
  - [vhodocker_init.yml (currently stale)](#vhodocker_inityml-currently-stale)
- [Roles that exist but aren't wired to an active playbook](#roles-that-exist-but-arent-wired-to-an-active-playbook)
- [Terraform (hetzner_terraform/)](#terraform-hetzner_terraform)

## Prerequisites

```
apt-get update
apt-get install -y ansible git
```

Vault password: `ansible.cfg` sets `vault_password_file = .vault_pass`, a
local, git-ignored file. Create it once so you don't need
`--ask-vault-pass` on every run:

```
echo 'your-vault-password' > .vault_pass && chmod 600 .vault_pass
```

(Never paste the actual password into a chat/AI session - type it directly
into the file yourself.)

## Playbook -> application quick reference

| Playbook | Application | Target (inventory group/host) | Key roles | Vault needed |
|---|---|---|---|---|
| `netbox_deploy.yml` | NetBox itself | `vhodockers` (vhohetzner1) | `netbox` | no |
| `netbox_discovery_deploy.yml` | Diode server + NetBox Diode plugin + Orb Agent | `vhodockers` (vhohetzner1) | `diode_server`, `netbox_diode_plugin`, `orb_agent` | yes - `host_vars/vhohetzner1/vault.yml` + `group_vars/Hetzner/vault.yml` |
| `update_hetzner.yml` | OS updates + SNMP agent for discovery | `Hetzner` (vhohetzner1 **and** vhohetzner2) | `common`, `snmp_agent` | yes - `group_vars/Hetzner/vault.yml` |
| `play-otrs.yml` | OTRS/Znuny ticketing | `otrs-p` | `otrs` | yes - `host_vars/otrs-p/vault.yml` |
| `bluecat_deploy.yml` | BlueCat BAM/BDDS lab | `lab3` (vholab3) | `bluecat` | no |
| `kvm_initial.yml` | Provision a new KVM guest from a cloud image | `localhost` (runs wherever invoked) | `kvm_provision_qcow2` or `kvm_provision_lvm` | no |
| `hostall_init.yml` | Baseline packages/updates for lab hosts | `hostall` (vholab3 currently) | `labcommon` | no |
| `vhoflask_init.yml` | Flask app (ACT v1.0) deploy | `vhohetzner1` | `flask` | no |
| `vhodocker_init.yml` | *(stale - see note below)* | `vhodocker` - **no such group exists** | `docker` | no |

## Inventory groups at a glance

From `inventory`:

| Group | Host(s) | Notes |
|---|---|---|
| `local` | `local_machine` (localhost) | |
| `hostall` | `vholab3` | vholab1/vholab2 rows are commented out |
| `lab1` | `vholab1` (`192.168.1.13`) | libvirt host; `group_vars/lab1` names its guest VMs `vhodocker`, `vhoflask` |
| `lab2` | `vholab2` (`192.168.0.12`) | libvirt host; guest VMs `vhoweb`, `vhopi` |
| `lab3` | `vholab3` (`192.168.178.32`) | BlueCat lab host |
| `vmlab1` | `react-lab` | |
| `vmlab2` | `vhoweb`, `vhopi` | |
| `Hetzner` | `vhohetzner1` (`188.245.69.219`), `vhohetzner2` (`195.201.27.165`) | both Hetzner Cloud VPS - see `hetzner_terraform/` |
| `vhodockers` | `vhohetzner1` | subset of `Hetzner`; runs the NetBox/Diode/Orb Docker stack |
| `otrs-p` | `otrs-p` (`192.168.102.217`) | |

`vhohetzner1` is a member of both `Hetzner` and `vhodockers` - anything in
`group_vars/Hetzner/` is visible to both `update_hetzner.yml` and
`netbox_discovery_deploy.yml` when they target it.

---

## Applications

### NetBox — `netbox_deploy.yml`

Deploys `netbox-docker` on `vhodockers` (vhohetzner1): NetBox itself,
Postgres, Redis, Redis-cache, and an nginx front door with Let's Encrypt via
certbot/Cloudflare DNS. See `roles/netbox/defaults/main.yml` for the FQDN,
bind port (`127.0.0.1:8083`), and certbot settings.

```
ansible-playbook -i inventory netbox_deploy.yml
```

### Diode + Orb Agent discovery (around NetBox) — `netbox_discovery_deploy.yml`

Layers NetBoxLabs' Diode ingestion pipeline, the NetBox Diode plugin, and
Orb Agent (network/SNMP discovery) **around** the existing NetBox install -
detects its current state at runtime and never touches `netbox_deploy.yml`'s
own files.

```
ansible-playbook -i inventory netbox_discovery_deploy.yml --ask-vault-pass
```

Useful tags: `--tags diode_server`, `--tags netbox_diode_plugin`
(alias `diode_plugin`), `--tags orb_agent`. `--skip-tags orb_agent` deploys
everything except Orb.

Order matters: `diode_server` runs before `netbox_diode_plugin` because the
plugin needs to know the real service name Diode's front door ends up with
(detected at runtime - see `roles/diode_server/tasks/detect_frontend.yml`),
and it creates the shared Docker network the plugin's compose overlay
attaches NetBox to.

**Don't add `--check` (dry run)** - this role is almost entirely `command`
tasks branching on `docker`/`docker compose` output (network checks, image
pulls, `up`/`ps`, container state parsing), which Ansible can't meaningfully
simulate. `command` tasks are skipped under `--check` by default, which
breaks the very first `register`-then-`when` chain in `preflight.yml` with
`'dict object' has no attribute 'rc'` - and the same pattern recurs
throughout `deploy.yml`. Use `--tags <role>` instead if you want a smaller
blast radius than the full playbook.

#### Before first run

- `cp host_vars/vhohetzner1/vault.yml.example host_vars/vhohetzner1/vault.yml`,
  fill in real secrets, `ansible-vault encrypt host_vars/vhohetzner1/vault.yml`.
- `orb_discovery_targets` in `group_vars/vhodockers.yml` defaults
  (role-level) to a deliberate non-routable placeholder (`192.0.2.0/24`) -
  never set `orb_dry_run: false` while it's still that placeholder (the
  `orb_agent` role asserts this and will fail the play if you try). On this
  host it's already been replaced with the real, permitted range
  (`10.0.0.0/24`) and `orb_dry_run` is `false` - see
  [Orb Agent discovery timing](#orb-agent-discovery--running-scans-timing-rescans)
  below for how it actually runs.
- `diode_nginx_bind_port` is `8280`, not the upstream default `8080` -
  `8080`/`8081` are already bound on this host by unrelated containers
  (`app-backend-1`/`app-frontend-1`), and `8180` (the first alternative
  tried) hit a Docker daemon bug where the in-memory port-allocator got
  stuck reporting "port is already allocated" after repeated rapid
  redeploys on that port during initial rollout, even with nothing actually
  listening - see the comment on `diode_nginx_bind_port` in
  `group_vars/vhodockers.yml`. Re-check with `ss -tlnp` if the host's other
  services have changed since.
- Disk space is tight on this host (observed 87.1% of 37.23GB used, ~4.8GB
  free) - `roles/diode_server/tasks/preflight.yml` gates on a minimum of 3GB
  free before pulling any images and will fail fast with a clear message if
  there isn't enough room; free up space first if it does.
- `diode_plugin_version` is pinned to `1.12.0`, not the latest release,
  because of an open NetBox-migration issue on the newest version - see
  `roles/netbox_diode_plugin/defaults/main.yml` for the tracking link before
  bumping it.
- New vault secrets as of the postgres/hydra env-var fix:
  `hydra_postgres_password` and `hydra_secrets_system_0` (>=32 chars) - add
  both to `host_vars/vhohetzner1/vault.yml` alongside the existing secrets.
- `cp group_vars/Hetzner/vault.yml.example group_vars/Hetzner/vault.yml`,
  fill in real (>=8 char) SNMPv3 passphrases,
  `ansible-vault encrypt group_vars/Hetzner/vault.yml` - required by both
  `update_hetzner.yml` (`roles/snmp_agent`, provisions the SNMPv3 user) and
  this playbook (`roles/orb_agent`'s `snmp_discovery` policy authenticates
  as that same user) - see
  [SNMP discovery agents](#snmp-discovery-agents-on-hetzner-hosts--update_hetzneryml)
  below before running.

#### Re-running / redeploying

- Every run of `netbox_discovery_deploy.yml` (or `--tags diode_server`) tears
  down and recreates the Diode server's own containers **and volumes**
  (`docker compose down --volumes --remove-orphans` in
  `/opt/netbox/discovery/diode`) before redeploying - see
  `roles/diode_server/tasks/reset.yml`. This is deliberate and safe: nothing
  in Diode's own postgres/redis is source-of-truth data (that's NetBox's
  job), it's just OAuth2 client state and reconciler dedup state, both fully
  reconstructable from `.env` / `oauth2/client/client-credentials.json` on
  the next `up`.
- This matters because Postgres only ever runs its
  `/docker-entrypoint-initdb.d` init scripts once, against an empty data
  directory. If a deploy fails partway through (e.g. a missing env var), the
  data volume is left initialized-but-incomplete - simply fixing the config
  and re-running `docker compose up -d` by hand will **not** retry that
  init script, so it stays broken forever without a clean volume. The
  automatic reset is what makes re-running the playbook after a config fix
  actually work.
- Set `diode_server_reset_on_deploy: false` in `group_vars/vhodockers.yml`
  once the stack is validated and stable, if you'd rather preserve Diode's
  state across future re-runs (e.g. you don't want to re-authenticate
  Orb/NetBox on every playbook run).
- To reset by hand without Ansible:
  `cd /opt/netbox/discovery/diode && docker compose down --volumes --remove-orphans`
- **Scope**: none of the above ever touches the actual NetBox containers
  (`netbox-docker-netbox-1`, `netbox-docker-netbox-worker-1`,
  `netbox-docker-postgres-1`, `netbox-docker-redis-1`,
  `netbox-docker-redis-cache-1` - the `netbox-docker` compose project
  deployed by `netbox_deploy.yml`/`roles/netbox`). Those hold NetBox's real
  inventory data and are never recreated or reset by
  `netbox_discovery_deploy.yml`.

If you need to fully wipe and redeploy **NetBox itself** (not the discovery
stack) - e.g. starting over from scratch - this is destructive and drops all
inventory data in NetBox's own database. `roles/netbox` has no backup step
of its own (the `netbox_backup_dir` used elsewhere in this repo is only for
the diode_server/netbox_diode_plugin roles' own config file backups -
`.env`, `plugins.py`, `client-credentials.json` - not NetBox's database), so
take one manually first:
`docker compose exec netbox /opt/netbox/netbox/manage.py dumpdata > backup.json`,
or a `pg_dump` of `netbox-docker-postgres-1`. Once you have a backup:

    cd /opt/netbox/netbox-docker
    docker compose down --volumes --remove-orphans
    # then re-run netbox_deploy.yml to recreate it from scratch

#### Orb Agent discovery — running scans, timing, rescans

- Discovery is scheduler-driven, not manually invoked. `orb_discovery_schedule`
  in `group_vars/vhodockers.yml` (default `*/15 * * * *`, every 15 min) is a
  cron expression the agent evaluates internally - there's no `orb-agent scan
  now` command in this deployment (`config_manager.active: local`, so there's
  no fleet manager API to poke either).
- On container start (deploy, restart, or recreate) it runs the policy
  immediately - confirmed in logs, `"policy applied successfully"` is
  followed within ~30s by `"running scanner"`. It then waits for the next
  cron tick for subsequent runs.
- `orb_discovery_timeout` (default 15 min, see
  `roles/orb_agent/defaults/main.yml`) is the max a single `network_discovery`
  scan is allowed to run before nmap is killed and the policy logs an error
  instead of results. Budget up to that long for a scan over a `/24` to
  finish, especially with `orb_discovery_skip_host: false` (ping-sweeps
  first, still touches every address).
- **Watch a run**: `docker logs -f orb-agent` on vhodockers - look for
  `"running scanner"` then either normal completion or a
  `"nmap scan timed out"` error.
- **Where results land**:
  - `orb_dry_run: true` - JSON files under
    `/opt/netbox/discovery/orb/data/dry-run/` on vhodockers, nothing sent
    anywhere else.
  - `orb_dry_run: false` (current setting) - sent to Diode over gRPC, which
    reconciles it into NetBox. Check NetBox's own Diode plugin menu
    (`127.0.0.1:8083`) for ingested/pending objects, not the filesystem.
- **Trigger a rescan on demand** (don't wait for the next cron tick): restart
  the container so it re-runs its start-of-life scan -
  `cd /opt/netbox/discovery/orb && docker compose restart orb-agent`. A full
  `--force-recreate` (or re-running `--tags orb_agent`, which now notifies a
  recreate handler on any `agent.yaml`/`.env` change) also works and is
  required anyway if you changed `orb_discovery_targets`, `schedule`, or any
  other policy field - see the note above `roles/orb_agent/handlers/main.yml`
  about `docker compose up -d` not detecting bind-mounted file content
  changes on its own.

### SNMP discovery agents on Hetzner hosts — `update_hetzner.yml`

`network_discovery` (nmap) only ever produces `IP Address` objects in
NetBox, with port/service data stuffed into a free-text `comments` field -
it can't model Devices, Interfaces, VLANs, or Prefixes. `snmp_discovery` is
a second Orb Agent backend/policy (in the same `agent.yaml`, deployed by
`netbox_discovery_deploy.yml`/`roles/orb_agent`) that can - but it needs an
SNMP agent actually running on the target hosts, which is what this
playbook provisions.

```
ansible-playbook -i inventory update_hetzner.yml
```

Runs on the whole `Hetzner` group (vhohetzner1 **and** vhohetzner2):
- `roles/common` - apt updates/upgrades, base packages, timesyncd config.
- `roles/snmp_agent` - installs `snmpd`, provisions an SNMPv3 user, and
  templates a minimal `snmpd.conf`. Must run (or at least have already run)
  before `netbox_discovery_deploy.yml --tags orb_agent`, since the Orb Agent
  policy authenticates against the user this role provisions.

Same schedule/timeout/rescan mechanics as `network_discovery` above, just
its own cron (`orb_snmp_schedule`, default every 6h - device/interface
inventory changes far less often than open ports) and its own timeout units
(`orb_snmp_timeout` is **seconds**, not the minutes `orb_discovery_timeout`
uses).

#### Security model (SNMPv3 authPriv only, deliberately not v1/v2c)

- No community string anywhere in this stack - v1/v2c sends its "password"
  in cleartext on every request, and community strings are routinely
  brute-forced when SNMP is reachable at all. SNMPv3 authPriv means every
  request is both authenticated (HMAC-SHA) and encrypted (AES), so even a
  full packet capture on the private network doesn't expose a replayable
  credential.
- `snmpd` binds explicitly to each host's private-network IP
  (`snmp_agent_bind_ip` in `host_vars/<host>/vars.yml`), never `0.0.0.0` -
  both hosts also carry public IPv4/IPv6 addresses
  (`hetzner_terraform/main.tf`), and SNMP has no legitimate reason to ever
  be reachable from the internet (it's a well-known reflection/
  amplification DDoS vector). This holds even if the firewall rule below
  were ever misconfigured or removed - defense in depth, not the only
  layer.
- `hetzner_terraform/firewall.tf` additionally scopes inbound UDP/161 to
  `10.0.0.3/32` (vhohetzner1's private IP) specifically, not the whole
  `10.0.0.0/24` - it's the only host that ever originates an SNMP query
  (Orb Agent runs there). Apply with the usual `terraform apply` in that
  directory before running this playbook, or the agent's SNMP queries will
  just time out.
- The provisioned user is **read-only** (`rouser ... priv` in
  `roles/snmp_agent/templates/snmpd.conf.j2`, no `rwuser`/write community
  anywhere) - it can observe device state but never change configuration.
- Auth/priv protocols are the classic SHA/AES-128 pair, not the stronger
  SHA256/AES256 Orb Agent's own client also supports - stock Ubuntu
  net-snmp builds don't reliably ship the extended AES-192/256 key
  localization draft, so assuming it works risks a broken deploy for
  marginal gain. SHA-1 is dated, but this is still authenticated +
  encrypted end to end, a world away from cleartext community strings.

#### Real-world ceiling

SNMP is poll-based (Orb Agent queries UDP/161 on each target; nothing
"broadcasts" to it) and only produces anything for hosts actually running
an SNMP agent. On this subnet
(`hcloud_network_subnet.network-subnet` in `hetzner_terraform/network.tf`,
`10.0.0.0/24`), `.1` is Hetzner's own auto-assigned private-network gateway
- not a `hcloud_server` resource in Terraform, not something we manage, and
it will never respond. Only `.2` (vhohetzner2) and `.3` (vhohetzner1) are
ours and can actually run `snmpd`.

### OTRS / Znuny — `play-otrs.yml`

Installs Znuny (OTRS fork) + Postgres + a nightly DB backup cron job on the
`otrs-p` host.

```
ansible-playbook -i inventory play-otrs.yml --ask-vault-pass -vvv
# quick connectivity check first:
ansible otrs-p -i inventory -m ping --ask-vault-pass
```

Version/paths pinned in `roles/otrs/defaults/main.yml`
(`znuny_version: "6.5.15"`, Postgres 15, backups to `/var/backups/otrs` at
midnight daily). Secrets (DB password, etc.) live in
`host_vars/otrs-p/vault.yml`.

### BlueCat Lab — `bluecat_deploy.yml`

Quick-creates a BlueCat BAM (Address Manager) lab VM via libvirt on
`vholab3` (`lab3` group).

```
ansible-playbook -i inventory bluecat_deploy.yml
```

Key vars (set in the playbook itself, override with `-e`):
- `number_of_bdds` - how many BDDS instances to deploy (default 3)
- `create_bdds` - set `false` to skip BDDS entirely and only deploy BAM
- `bridge_nw` - `vhobr3-network` (default here) or `vhobr1-network`

Two networking modes:
- `vhobr1-network` - BAM and BDDS share the host's own network; everything
  is reachable on the host network directly, no routing/forwarding needed.
- `default` (libvirt's own NAT network) - BAM/BDDS sit behind virsh's
  default network and need another KVM running an nginx reverse-proxy
  (`roles/nginx-bam`, currently commented out in this playbook - see
  [below](#roles-that-exist-but-arent-wired-to-an-active-playbook)) to
  reach them. Useful if the routing KVM has a public IP; requires that KVM
  to be on the same subnet as the BAM host.

### KVM provisioning — `kvm_initial.yml`

Provisions a new KVM guest from a cloud image, either on qcow2 or an LVM
volume. Runs against `localhost` (i.e. on whichever KVM hypervisor host you
invoke it from).

Directly from this repo:

```
ansible-playbook -i inventory kvm_initial.yml \
  -e "provision_method=qcow2 vm_name=qcow_1 virt_network=vhobr3-network qcow2_size=10G target_distro=debian"
```

Or via `ansible-pull` (as documented historically, pulling from a mirror
repo that vendors this same playbook):

```
# qcow2
ansible-pull -U https://github.com/Dispelk9/ansible-pull-docker.git kvm_initial.yml \
  -e "provision_method=qcow2 vm_name=qcow_1 virt_network=vhobr3-network qcow2_size=10G target_distro=debian" -v

# LVM
ansible-pull -U https://github.com/Dispelk9/ansible-pull-docker.git kvm_initial.yml \
  -e "provision_method=lvm vm_name=lvm_1 virt_network=vhobr3-network lvm_disk_size=10G lvm_vg_name=vgubuntu target_distro=debian" -v
```

`target_distro` can be `debian` or `ubuntu`; adjust `vm_name`,
`virt_network`, and the qcow2/LVM size vars per guest. Defaults for the rest
live in `roles/kvm_provision_qcow2/defaults/main.yml` and
`roles/kvm_provision_lvm/defaults/main.yml`.

### Lab hosts bootstrap — `hostall_init.yml`

Baseline packages/updates (apt upgrade, `fail2ban`, `git`, `ansible`,
`openssh-server/client`) for lab hosts, currently just `vholab3`
(`hostall` group). Per-lab blocks for `vholab1`/`vholab2` and the
`vhoweb`/`vhopi` VMs are present but commented out - see
[below](#roles-that-exist-but-arent-wired-to-an-active-playbook).

```
ansible-playbook -i inventory hostall_init.yml
```

### Flask app (ACT) on vhohetzner1 — `vhoflask_init.yml`

Deploys the ACT v1.0 Flask application to **vhohetzner1** (the Docker host -
see the `#Docker include ACT / Certcheck / Checkmk` comment on
`hcloud_server.vho_hetzner1` in `hetzner_terraform/main.tf`; vhohetzner2 is
the mail server, not this one).

```
ansible-playbook -i inventory vhoflask_init.yml
```

`roles/flask/defaults/main.yml` expects a deploy key at
`/root/.ssh/github_terra` on the target. Certbot/SSL setup and actually
starting the app are manual steps noted as comments at the top of
`roles/flask/tasks/main.yml` (install certbot via snap, `certbot --apache`,
`pip install pyopenssl --upgrade`, then run with
`nohup python3 adduct_flask.py > /tmp/flask_log.txt 2>&1 &`) - this role
only prepares the host, it doesn't start the app itself.

### `vhodocker_init.yml` (currently stale)

Targets `hosts: vhodocker` (singular) with `roles/docker` - but the current
`inventory` file has no group or host named `vhodocker`, only `vhodockers`
(plural, a different group containing vhohetzner1). `group_vars/lab1.yml`
does define a libvirt guest VM *named* `vhodocker` on `vholab1`, but that's
a VM name for the `vholab1`/`vholab2` roles' `community.libvirt.virt`
tasks, not an Ansible inventory entry Docker tasks could run *inside*.

Running this playbook as-is against the current `inventory` will fail with
"no hosts matched" unless you pass `-l`/`-i` pointing at a real target, or
add a matching `vhodocker` entry to `inventory` first (e.g. once that
libvirt VM has its own reachable IP). Flagging this rather than guessing -
fix the inventory/playbook mismatch before relying on it.

---

## Roles that exist but aren't wired to an active playbook

These have real task files under `roles/` but every playbook that would
invoke them currently has that block commented out - they won't run from
any of the commands above without first uncommenting something:

| Role | Would be invoked from | What it does |
|---|---|---|
| `apache` | `hostall_init.yml` (commented `vhoweb` block) | Installs apache2, clones/deploys the `dispelk9.github.io` static site |
| `pihole` | `hostall_init.yml` (commented `vhopi` block) | `pihole -g` / `pihole -up` - gravity update + upgrade |
| `nginx-bam` | `bluecat_deploy.yml` (commented block, `bridge_nw == "default"` case) | nginx reverse-proxy in front of BlueCat BAM when it's on libvirt's default NAT network |
| `vholab1` | `hostall_init.yml` (commented block) | Starts/autostarts the libvirt VMs listed in `group_vars/lab1` (`vhodocker`, `vhoflask`) on `vholab1` |
| `vholab2` | `hostall_init.yml` (commented block) | Same, for `group_vars/lab2`'s VMs (`vhoweb`, `vhopi`) on `vholab2` |
| `management` | not referenced by any playbook | One task: keeps `SSH_AUTH_SOCK` in sudoers' `env_keep` so `git clone` over a forwarded agent works under `sudo` |

## Terraform (`hetzner_terraform/`)

A sibling directory (separate tool, not Ansible) that provisions the two
Hetzner Cloud VPS themselves (`vho_hetzner1`/`vho_hetzner2`), their shared
private network (`10.0.0.0/24`), and their Cloud Firewalls. Ansible
configures what runs *on* these hosts; Terraform provisions the hosts and
network/firewall rules around them. State is remote (Terraform Cloud,
workspace `VHO-Cloud`) - `admin_ips`/`cloudflare_ips` in `variables.tf` may
be overridden there rather than by the local default.

```
cd hetzner_terraform
terraform apply
```
