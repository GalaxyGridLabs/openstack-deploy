# ceph setup
Following https://docs.ceph.com/projects/ceph-ansible/en/latest/

```bash
# Dev container

```


# openstack-deploy
Following https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html
But more this https://docs.openstack.org/kolla-ansible/latest/user/multinode.html


```bash
# Devcontainer
sudo apt update && sudo apt install -y git python3-dev libffi-dev gcc libssl-dev python3-venv
source ./venv/bin/activate
cd openstack
pip install -U pip
pip install 'ansible-core>=2.16,<2.17.99'

pip install git+https://opendev.org/openstack/kolla-ansible@master
pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/master

sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla

cp -r ../venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
cp ../venv/share/kolla-ansible/ansible/inventory/multinode .

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

# map OIDC attributes -> keystone attributes
openstack mapping create --rules ./mapping-rules.json vault_mapping
# Create openid protocol for vault provider
openstack federation protocol create openid --mapping vault_mapping --identity-provider vault

# Create group, project, and role for admins
openstack group create admin
openstack role add --group admin --project admin admin

# Create group, project, and role for red team users
openstack group create red_teamers
openstack project create red_team
openstack role add --group red_teamers --project red_team member


# Create base networking - admin
    source ~/admin-openrc.sh
openstack network create --external --provider-physical-network physnet1 \
    --share --provider-network-type flat public1

openstack subnet create --no-dhcp --ip-version 4 --project red_team \
    --allocation-pool "start=10.0.2.10,end=10.0.2.250" --network public1 \
    --subnet-range 10.0.2.0/24 --gateway 10.0.2.1 public1-subnet

openstack router create internal1

openstack network create internal1

openstack subnet create --ip-version 4 \
    --subnet-range 10.0.0.0/24 --network internal1 \
    --gateway 10.0.0.1 --dns-nameserver 8.8.8.8 \
    internal1

openstack router add subnet internal1 internal1

openstack router set --external-gateway public1 internal1

# Create red team network
source ~/red_team-openrc-admin.sh
openstack router create red_team

openstack network create red_team

openstack subnet create --ip-version 4 \
    --subnet-range 10.0.3.0/24 --network red_team \
    --gateway 10.0.3.1 --dns-nameserver 8.8.8.8 \
    red_team

openstack router add subnet red_team red_team

openstack router set --external-gateway public1 red_team

openstack server create \
    --image cirros \
    --flavor m1.tiny \
    --network red_team \
    --security-group default \
    red_team-test
```
