# ansible-vholab<br />
# Welcome to vho Lab Ansible configuration<br />
apt-get update<br />
apt get -install -y ansible git<br />

target_distro can be debian or ubuntu<br />
with adjustment of vm name virt_network and size of qcow or lvm<br />



# For qcow2
ansible-pull -U https://github.com/Dispelk9/ansible-pull-docker.git kvm_initial.yml -e "provision_method=qcow2 vm_name=qcow_1 virt_network=vhobr3-network qcow2_size=10G target_distro=debian" -v

# For LVM
ansible-pull -U https://github.com/Dispelk9/ansible-pull-docker.git kvm_initial.yml -e "provision_method=lvm vm_name=lvm_1 virt_network=vhobr3-network lvm_disk_size=10G lvm_vg_name=vgubuntu target_distro=debian" -v


playdocker<br />


playflask<br />
pushing flask application for ACTv1.0 to vhohetzner2<br />

playlab<br />
roll all changes to hosts/VMs in vholab network<br />

###
netbox_discovery_deploy.yml<br />
deploys NetBoxLabs Diode server + NetBox Diode plugin + Orb Agent around the
existing NetBox install on vhodockers (deployed by netbox_deploy.yml /
roles/netbox). Never touches the netbox role's own files.<br />

    ansible-playbook -i inventory netbox_discovery_deploy.yml --ask-vault-pass

Don't add `--check` (dry run) - this role is almost entirely `command`
tasks branching on `docker`/`docker compose` output (network checks,
image pulls, `up`/`ps`, container state parsing), which Ansible can't
meaningfully simulate. `command` tasks are skipped under `--check` by
default, which breaks the very first `register`-then-`when` chain in
`preflight.yml` with `'dict object' has no attribute 'rc'` - and the same
pattern recurs throughout `deploy.yml`. Use `--tags <role>` to scope a run
instead if you want a smaller blast radius than the full playbook.

Before first run:
- `cp host_vars/vhohetzner1/vault.yml.example host_vars/vhohetzner1/vault.yml`,
  fill in real secrets, `ansible-vault encrypt host_vars/vhohetzner1/vault.yml`.
- `orb_discovery_targets` in `group_vars/vhodockers.yml` is a deliberate
  non-routable placeholder (`192.0.2.0/24`) - replace with the real
  network(s) you have permission to scan before setting `orb_dry_run: false`
  (defaults to `true`: Orb Agent writes discovered inventory to
  `/opt/netbox/discovery/orb/data/dry-run/` instead of calling Diode).
- `diode_nginx_bind_port` is `8180`, not the upstream default `8080` -
  `8080`/`8081` are already bound on this host by unrelated containers
  (`app-backend-1`/`app-frontend-1`). Re-check with `ss -tlnp` if the host's
  other services have changed since.
- Disk space is tight on this host (observed 87.1% of 37.23GB used, ~4.8GB
  free) - `roles/diode_server/tasks/preflight.yml` gates on a minimum of 3GB
  free before pulling any images and will fail fast with a clear message if
  there isn't enough room; free up space first if it does.
- `diode_plugin_version` is pinned to `1.12.0`, not the latest release,
  because of an open NetBox-migration issue on the newest version - see
  `roles/netbox_diode_plugin/defaults/main.yml` for the tracking link before
  bumping it.
- New vault secrets as of the postgres/hydra env-var fix below:
  `hydra_postgres_password` and `hydra_secrets_system_0` (>=32 chars) - add
  both to `host_vars/vhohetzner1/vault.yml` alongside the existing secrets
  before running.

Re-running / redeploying:
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
  directory. If a deploy fails partway through (e.g. a missing env var, like
  the one that caused `diode-postgres-1` to go unhealthy before this fix),
  the data volume is left initialized-but-incomplete - simply fixing the
  config and re-running `docker compose up -d` by hand will **not** retry
  that init script, so it stays broken forever without a clean volume. The
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

If you do need to fully wipe and redeploy NetBox itself (not the discovery
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

###
playbc<br />
quick create Bluecat Lab on your debian machine<br />
number_of_bdds: can be changed <br />
two modes:<br />
- vhobr1-network if the BAM and BDDS should be accessing the same network as the host.<br />
    pro: this mode then all BAM and BDDSs can be accessible on host network, no routing and forwarding required.<br />
- default if the BAM and BDDS should be using virsh default network and be acessible using another kvm with nginx-reverse-proxy using docker container.<br />
    pro: can be used if the routing kvm has public IP<br />
    con: this mode required a kvm on same subnet as host<br />

