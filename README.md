# ansible-pull-docker
# testing ansible pull inside a debian docker
apt-get update
apt get -install -y ansible git
ansible-pull -U https://github.com/Dispelk9/ansible-pull-docker.git

