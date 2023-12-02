# ansible-pull-docker
# testing ansible pull inside a debian docker
apt-get update\n
apt get -install -y ansible git\n
ansible-pull -U https://github.com/Dispelk9/ansible-pull-docker.git test_pull.yml

