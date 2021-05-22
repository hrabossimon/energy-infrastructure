# Virtualized Energy Inrasctructure using KYPO and OpenStack platforms
This sandbox is used to demonstate functionality of energy infrastructure using KYPO and OpenStack platforms.
For more informations navigate to OpenStack [kolla-ansible][KOLLA] GitHub repository and [KYPO][KYPO-GITLAB] GitLab repositories.

## Vagrant virtual environment
Install dependencies.
```sh
sudo apt install vagrant ruby-dev ruby-libvirt python-libvirt qemu-utils qemu-kvm libvirt-dev nfs-kernel-server zlib1g-dev gcc git
vagrant plugin install --plugin-version ">= 0.0.31" vagrant-libvirt
vagrant plugin install vagrant-hostmanager
vagrant plugin install vagrant-vbguest
```
Go to directory containing your Vagrantfile and run.
```sh
vagrant up
vagrant ssh operator
```

---
**NOTE**
This guide was tested with 28GB ram, libvirt provider and ubuntu bionic distro.
---

## OpenStack installation
Update and upgrade your system packages.
```sh
sudo apt update
sudo apt upgrade
```

Install packages.
```sh
sudo apt install python3-dev python3-venv libffi-dev gcc libssl-dev git
```

Create and avtivate virtual environment.
```sh
python3 -m venv $HOME/kolla-openstack
source $HOME/kolla-openstack/bin/activate
```

---
**NOTE**
Don't leave virtual environment.
---

Install pip packages.
```sh
pip install -U pip
pip install 'ansible<2.10'
pip install kolla-ansible
```

Create ansible configuration to speed up installation.
```sh
vim $HOME/ansible.cfg
```
```text
[defaults]
host_key_checking=False
pipelining=True
forks=100
```

Configure Kolla-ansible for all-in-one OpenStack deployment.
```sh
sudo mkdir /etc/kolla
sudo chown $USER:$USER /etc/kolla
cp $HOME/kolla-openstack/share/kolla-ansible/etc_examples/kolla/* /etc/kolla/
cp $HOME/kolla-openstack/share/kolla-ansible/ansible/inventory/all-in-one .
```

Define global deployment options.
```sh
vim /etc/kolla/globals.yml
```

Following options were used for this environment.
```yaml
---
config_strategy: "COPY_ALWAYS"
kolla_base_distro: "ubuntu"
kolla_install_type: "source"
openstack_release: "victoria"
kolla_internal_vip_address: "172.28.128.254"
network_interface: "eth1"
neutron_external_interface: "eth2"
neutron_plugin_agent: "linuxbridge"
enable_heat: "{{ enable_openstack_core | bool }}"
```

Generate passwords.
```sh
kolla-genpwd
```

Deploy OpenStack.

---
**NOTE**
Remove unwanted records in your /ets/hosts file i.e. 127.0.2.1 record.
---
```sh
kolla-ansible -i all-in-one bootstrap-servers
kolla-ansible -i all-in-one prechecks
kolla-ansible -i all-in-one deploy
```

Run post deployment tasks.
```sh
pip install python-openstackclient python-neutronclient python-glanceclient
kolla-ansible post-deploy
source /etc/kolla/admin-openrc.sh
```
Edit init-runonce file so it suits to your environment.
```sh
vim kolla-openstack/share/kolla-ansible/init-runonce
```
Edited values for this guide.
```yaml
ENABLE_EXT_NET=${ENABLE_EXT_NET:-1}
EXT_NET_CIDR=${EXT_NET_CIDR:-'192.168.122.0/24'}
EXT_NET_RANGE=${EXT_NET_RANGE:-'start=192.168.122.15,end=192.168.122.45'}
EXT_NET_GATEWAY=${EXT_NET_GATEWAY:-'192.168.122.1'}

if ! $KOLLA_OPENSTACK_COMMAND flavor list | grep -q m1.tiny; then
    $KOLLA_OPENSTACK_COMMAND flavor create --id 1 --ram 512 --disk 1 --vcpus 1 standard.tiny
    $KOLLA_OPENSTACK_COMMAND flavor create --id 2 --ram 2048 --disk 10 --vcpus 1 standard.small
    $KOLLA_OPENSTACK_COMMAND flavor create --id 3 --ram 4096 --disk 10 --vcpus 2 standard.medium
    $KOLLA_OPENSTACK_COMMAND flavor create --id 4 --ram 8192 --disk 30 --vcpus 4 standard.large
    $KOLLA_OPENSTACK_COMMAND flavor create --id 5 --ram 16384 --disk 160 --vcpus 8 standard.xlarge
fi
```
Run init-runonce script.
```sh
kolla-openstack/share/kolla-ansible/init-runonce
```

## KYPO installation
Install KYPO using official [installation guide][KYPO].

---
**NOTE**
You may need to wait several minutes to setup SSH access to nodes after creating KYPO infrastructure.
---


After KYPO deployment you need to setup connection between KYPO Head and OpenStack identity service.
Identify router with same network like you KYPO Head instance
```sh
sudo ip netns exec qrouter-ID ip a
```

Add route with subnet of your OpenStack management interface.
```sh
sudo ip netns exec qrouter-ID ip route add <management-subnet> via <provider-IP-address>
```

## Energy Infrastructure
Add your sandbox definition publiched in Git repository to internal KYPO git.
Run following from KYPO Head node:
```sh
git clone -q --bare https://github.com/hrabossimon/energy-infrastructure.git
docker cp energy-infrastructure.git git-internal-ssh:/repos/prototypes-and-examples/sandbox-definitions
```

Navigate to KYPO portal web page and login with credentials defined in your extra-vars.yml file.
KYPO portal can be found on page https://<kypo_crp_head> where <kypo_crp_head> is IP address of your KYPO Head node.
Go to **Definitions** -> **Create** and add Git repository and revision.
*Git URL: git@git-internal-ssh:/repos/prototypes-and-examples/sandbox-definitions/energy-infrastructure.git*
*Revision: master*

Go to **Pool** -> **Create** and add your sandbox definition
Click **Allocate sandboxes** and wait for sandbox creation

---
**NOTE**
You can watch progress by clicking Pool title and Request
---

## Verify functionality

After sandbox creation download SSH access in **Pool** page.
Unzip and copy files to your .ssh directory.
```sh
unzip ssh-access.zip -d ~/.ssh/
```

Access to Energy Infrastructure nodes.
```sh
ssh -F ~/.ssh/pool-id-ID-sandbox-id-ID-management-config concentrator
ssh -F ~/.ssh/pool-id-ID-sandbox-id-ID-management-config smart-meter
```

Install DLMS server on smart-meter node.
```sh
git clone https://github.com/Gurux/gurux.dlms.java.git
cd gurux.dlms.java/gurux.dlms.server.example2.java/
mvn clean package
java -jar target/gurux.dlms.server.example2.java-0.0.1-SNAPSHOT.jar
```

Download data from concentrator node.
```sh
git clone https://github.com/Gurux/gurux.dlms.java.git
cd gurux.dlms.java/gurux.dlms.client.example.java/
mvn clean package
java -jar target/gurux.dlms.client.example.java-0.0.1-SNAPSHOT.jar -h <smart-meter-IP-address> -p 4061
```



[KYPO]: <https://docs.crp.kypo.muni.cz/installation-guide/installation-guide-overview/>
[KOLLA]: <https://github.com/openstack/kolla-ansible>
[KYPO-GITLAB]: <https://gitlab.ics.muni.cz/muni-kypo-crp>