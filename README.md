# openstack-deploy
Following https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html
But more this https://docs.openstack.org/kolla-ansible/latest/user/multinode.html


```bash
# Devcontainer
sudo apt update && sudo apt install -y git python3-dev libffi-dev gcc libssl-dev python3-venv
source ./venv/bin/activate
pip install -U pip
pip install 'ansible-core>=2.16,<2.17.99'

pip install git+https://opendev.org/openstack/kolla-ansible@master
pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/master

sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla

cp -r ./venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
cp ./venv/share/kolla-ansible/ansible/inventory/multinode .

kolla-ansible install-deps

kolla-genpwd

cp /etc/kolla/globals.yml .
vi ./globals.yml
vi ./multinode

cp ./globals.yml /etc/kolla/globals.yml

kolla-ansible bootstrap-servers -i ./multinode
kolla-ansible prechecks -i ./multinode
kolla-ansible deploy -i ./multinode
kolla-ansible validate-config -i ./multinode
```

# Todo
