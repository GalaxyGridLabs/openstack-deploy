# ceph setup
Following https://docs.ceph.com/en/latest/cephadm/install/

## Setup OS
```bash
# Configure DHCP
# HPs
sudo dhcpcd enp4s0f1 # neutron
sudo dhcpcd ens6f0 #10gig

# Dell
sudo dhcpcd eno3 # neutron
sudo dhcpd

```

## Bootstrap Ceph
```bash
# compute-83
CEPH_RELEASE=17.2.7 # Newest version that doesn't have issues with glibc
curl --silent --remote-name --location https://download.ceph.com/rpm-${CEPH_RELEASE}/el9/noarch/cephadm
chmod +x ./cephadm

./cephadm add-repo --release quincy
./cephadm install ceph-common
# Get TLS cert from Vault - Cronjob to get update cert from vault
# Add ceph user with sudo NOPASSWD to all hosts
# Generate signed SSH cert
./cephadm bootstrap --mon-ip 10.10.254.83 --dashboard-key ?? --dashboard-crt ?? --skip-ssh --ssh-signed-cert ?? --ssh-private-key ~/.ssh/id_ed --ssh-user ceph
ceph orch host add compute-82 10.10.254.82 --labels _admin
ceph orch host add compute-83 10.10.254.81 --labels _admin

# Fix bug with ceph and app armor on each host https://www.reddit.com/r/ceph/comments/1g6od5x/having_issues_getting_a_ceph_cluster_off_the/
sudo ln -s /etc/apparmor.d/MongoDB_Compass /etc/apparmor.d/disable/
sudo apparmor_parser -R /etc/apparmor.d/MongoDB_Compass

# Rescan each host - may not be necesarry might just be a waiting game.
ceph orch host rescan compute-81 --with-summary
ceph orch host rescan compute-82 --with-summary
ceph orch host rescan compute-82 --with-summary

# Verify the disks we expect are present
ceph orch device ls --wide --refresh

# Add the available disks
ceph orch apply osd --all-available-devices
```

## Setup ceph pool
https://docs.ceph.com/en/latest/rbd/rbd-openstack/

```bash
create_rbd_pool() {
    ceph osd pool create $1 replicated
    ceph osd pool application enable $1 rbd
    ceph osd pool set $1 size 2
    ceph osd pool set $1 min_size 1
    rbd pool init $1
}
create_rbd_pool volumes
create_rbd_pool images
create_rbd_pool backups
create_rbd_pool vms


# Create users and creds for openstack services
#### REMEMBER: remove the leading `\t` from the key file it breaks kolla's ini parser.
ceph auth get-or-create client.glance mon 'profile rbd' osd 'profile rbd pool=images' mgr 'profile rbd pool=images'
ceph auth get-or-create client.cinder mon 'profile rbd' osd 'profile rbd pool=volumes, profile rbd pool=vms, profile rbd-read-only pool=images' mgr 'profile rbd pool=volumes, profile rbd pool=vms'
ceph auth get-or-create client.cinder-backup mon 'profile rbd' osd 'profile rbd pool=backups' mgr 'profile rbd pool=backups'

[client.cinder-backup]
	key = AQB[...CLIP...]8i+nw==
```

**The keyrings should look like the following**
```bash
vscode ➜ /workspaces/openstack-deploy (setup) $ tree ./openstack/config/
./openstack/config/
├── cinder
│   ├── ceph.conf
│   ├── cinder-backup
│   │   ├── ceph.client.cinder-backup.keyring
│   │   └── ceph.client.cinder.keyring
│   └── cinder-volume
│       └── ceph.client.cinder.keyring
├── glance
│   ├── ceph.client.glance.keyring
│   ├── ceph.conf
│   └── glance.conf
├── neutron
│   └── ml2_conf.ini
└── nova
    ├── ceph.client.cinder.keyring
    ├── ceph.client.nova.keyring
    └── ceph.conf
```

- `nova/ceph.client.cinder.keyring` == `nova/ceph.client.nova.keyring`
- `glance/ceph.client.glance.keyring` is unique
- `ceph.client.cinder.keyring` and `ceph.client.cinder-backup` are shared in a few places.



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

# Create group, project, and role for lab users
openstack group create lab-users
openstack project create lab-users
openstack role add --group lab-users --project lab-users admin

# Create group, project, and role for red team users
openstack group create red_teamers
openstack project create red_team
openstack role add --group red_teamers --project red_team member

# Verify ceph setup - list cirros image
root@compute-81:~# rbd ls images
697c2666-3ec8-4b89-9536-9863d8098a79
root@compute-81:~# rbd info images/697c2666-3ec8-4b89-9536-9863d8098a79
rbd image '697c2666-3ec8-4b89-9536-9863d8098a79':
	size 112 MiB in 14 objects
	order 23 (8 MiB objects)
	snapshot_count: 1
	id: e1f5cb691e70
	block_name_prefix: rbd_data.e1f5cb691e70
	format: 2
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	op_features:
	flags:
	create_timestamp: Mon Dec 30 04:08:11 2024
	access_timestamp: Mon Dec 30 04:08:11 2024
	modify_timestamp: Mon Dec 30 04:08:11 2024


# Create base networking - admin
source ~/admin-openrc.sh
# openstack network create --external --provider-physical-network physnet1 \
#     --share --provider-network-type vlan \
#     --provider-segment 8 public1

openstack network create --external --provider-physical-network physnet1 \
    --share --provider-network-type flat public1



openstack subnet create --no-dhcp --ip-version 4 \
    --allocation-pool "start=10.10.12.10,end=10.10.12.230" --network public1 \
    --subnet-range 10.10.12.0/24 --gateway 10.10.12.1 public1-subnet


openstack server create \
    --image cirros \
    --flavor m1.tiny \
    --network public1 \
    --security-group default \
    net-test


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
