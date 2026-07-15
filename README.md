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

    ansible-playbook -i inventory netbox_discovery_deploy.yml --ask-vault-pass --check   # dry run first
    ansible-playbook -i inventory netbox_discovery_deploy.yml --ask-vault-pass

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

