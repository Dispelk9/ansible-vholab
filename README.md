# ansible-vholab<br />
# ansible pull inside a debian/ubuntu to quick create a KVM<br />
apt-get update<br />
apt get -install -y ansible git<br />

to pull a new Ubuntu KVM using ansible-pull<br />
ansible-pull -U https://github.com/Dispelk9/ansible-pull-docker.git kvm_initial.yml<br />


playflask<br />
local pushing flask application for ACTv1.0 to hetzner VM<br />

playlab<br />
roll all changes to hosts/VMs in vholab network<br />
