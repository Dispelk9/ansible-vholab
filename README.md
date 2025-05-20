# ansible-vholab<br />
# testing ansible pull inside a debian/ubuntu<br />
apt-get update<br />
apt get -install -y ansible git<br />

to pull a new Ubuntu KVM using ansible-pull<br />
ansible-pull -U https://github.com/Dispelk9/ansible-pull-docker.git kvm_initial.yml<br />


playdocker<br />


playflask<br />
pushing flask application for ACTv1.0 to vhohetzner2<br />

playlab<br />
roll all changes to hosts/VMs in vholab network<br />

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

