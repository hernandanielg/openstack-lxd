# Openstack on LXD

## Prerequisites

### Hardware Requirements
* CPU with Virtualization capabilities (Intel VT or AMD-V)
* 4 GB of RAM
### OS Requirements
* Ubuntu Focal 20.04 LTS kernel 5.4.0
* NOTE: LXD supports minimum kernel version 3.13
### Software Requirements
* Juju version 2.9.14
* LXD version 4.0.7

This document is to deploy Openstack Wallaby version over LXD containers using Juju as model-operator manager


## Setup LXD

First we need to initialize LXD
```
lxd init --auto
```

As juju does not support IPv6 we need to disable IPv6 on LXD
```
lxc network unset lxdbr0 ipv6.address
lxc network unset lxdbr0 ipv6.nat
```

## Setup Juju

Let’s install juju on Ubuntu focal 20.04 LTS using snap
```
sudo snap install juju --classic --channel=2.9/stable
```

Bootstrap a Juju controller on LXD
```
juju bootstrap localhost lxd
```

Verify that Juju controller is working correctly checking its status
```
juju status
```

If no error message we are ready to continue


## LXC Profile

For simplicity we’ll modify the default juju model LXC profile
```
cd openstack-lxd/
cat lxd-profile.yaml | lxc profile edit juju-default
```

If using a different model name change LXC profile as `juju-<model>`

**NOTE:** this profile is applied to all containers in the same juju model

## Deploy Openstack

To deploy Openstack on LXD use the juju client and deploy the `bundle.yaml` file
```
juju deploy ./bundle.yaml
```

This process might take some time be sure to wait until most of the units have finished installing packages and are ready before continuing to the next step

## Vault initialization

NOTE: wait until keystone is ready before performing vault initialization

We need to install vault client tool
```
sudo snap install vault
```

Replace `<vault IP>` with the vault unit IP that can be obtained from juju status output
```
export VAULT_ADDR="http://<vault IP>:8200"
```

Proceed to initialization
```
vault operator init -key-shares=5 -key-threshold=3
```

From previous command use the keys to unseal vault operator
```
vault operator unseal key1
vault operator unseal key2
vault operator unseal key3
```

Then authorize the charm using the Initial Root Token
```
export TOKEN=<Initial Root Token>
juju run-action --wait vault/leader authorize-charm token=$TOKEN
```

Generate Root CA certificate
```
juju run-action --wait vault/leader generate-root-ca
```

## Setup Openstack

At this point we should have a Openstack ready to use


We need to perform some configurations in order to set up Openstack
*  Import an image
* Setup networks
* Create an Openstack flavor
* Create and import a keypair
* Open ports in Security Groups

Install Openstack CLI Client
```
sudo snap install openstackclients --channel=wallaby/stable
```

Install `jq`
```
sudo apt install jq --yes
```

Get the root CA certificate from Vault and put it in a `root_ca.pem` file
```
juju run-action --wait vault/leader get-root-ca --format json \
  | jq -r .[].results.output > root_ca.pem
```

Load Openstack environment variables values
```
export _root_ca=root_ca.pem
source novarc
```

Image import
```
export IMG_URL=http://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img

curl $IMG_URL | openstack image create --public \
  --container-format=bare --disk-format=qcow2 focal-amd64
```

Create and configure the external network
```
openstack network create ext_net --external --share \
   --default --provider-network-type flat \
   --provider-physical-network physnet1

export SUBNET=$(lxc network get lxdbr0 ipv4.address)
export GATEWAY=$(echo $SUBNET | cut -d/ -f1)

openstack subnet create ext_subnet \
   --subnet-range $SUBNET \
   --gateway $GATEWAY \
   --network ext_net
```

Create internal network
```
openstack network create int_net --internal

export DNS=$(echo $SUBNET | cut -d/ -f1)

openstack subnet create int_subnet \
   --allocation-pool start=192.168.20.10,end=192.168.20.99 \
   --subnet-range 192.168.20.0/24 \
   --gateway 192.168.20.1 --dns-nameserver $DNS \
   --network int_net
```

Create router and attach it to both networks
```
openstack router create router1
openstack router add subnet router1 int_subnet
openstack router set router1 --external-gateway ext_net
```

Create a flavor “m1.tiny” with small resources limitations
```
openstack flavor create --public --ram 512 --disk 5 --ephemeral 0 --vcpus 1 m1.tiny
```

Create and import SSH key
```
ssh-keygen -q -N '' -f ~/.ssh/id_mykey

openstack keypair create --public-key ~/.ssh/id_mykey.pub mykey
```

Configure security groups
```
for i in $(openstack security group list | awk '/default/{ print $2 }'); do
    openstack security group rule create $i --protocol icmp --remote-ip 0.0.0.0/0;
    openstack security group rule create $i --protocol tcp --remote-ip 0.0.0.0/0 --dst-port 22;
done
```

## Testing the deployment

### Manually

Using openstack client proceed and create a new instance
```
NET_ID=$(openstack network show int_net -f value -c id)

openstack server create --image focal-amd64 \
    --flavor m1.tiny --key-name mykey \
    --network=$NET_ID focal-1
```

Assign a new floating IP to the newly created instance
```
FLOATING_IP=$(openstack floating ip create -f value \
   -c floating_ip_address ext_net)

openstack server add floating ip focal-1 $FLOATING_IP
```

Access the machine using SSH
```
ssh -i ~/.ssh/id_mykey ubuntu@$FLOATING_IP
```

Wait until server is running and SSH service is running and listening to new connections
