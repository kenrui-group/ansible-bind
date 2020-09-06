# Overview
Install or upgrade to latest BIND minor version and apply zone files.  This playbook assumes the server 
has already been set with the correct network details (eg IP, DNS servers, etc).

# Features
The playbook is idempotent and permits a regular rebuild of DNS servers to achieve the following:
* Install the latest BIND available.
* Update of zone records.
* Rotate RNDC keys.

Playbook also comes with settings for testing using Molecule.

# Configuration
Following variables are configurable in vars/main.yml.  Changes need to be synced to molecule/default/roles/test_bind/vars/main.yml.
* BIND major version to install or upgraded to.
* Domain name (eg mydomain.com).
* Hostnames and IP addresses.

DNS servers' IPs are configured in the inventory-dns file.

# Reference
* https://www.digitalocean.com/community/tutorials/how-to-setup-dnssec-on-an-authoritative-bind-dns-server--2
* https://blog.apnic.net/2019/05/23/how-to-deploying-dnssec-with-bind-and-ubuntu-server/

# DNS Check Tools
* https://tools.dnsstuff.com
* https://mxtoolbox.com/DnsLookup.aspx

# BIND Version
Reference following site for latest stable version.
* https://kb.isc.org/docs/aa-00896

# Example Usage 
## Deployment To Production Servers
### Both Master and Slave
```bash
ansible-playbook -i inventory-dns bind.yml -k
```
### Master - ns1.mydomain.com
```bash
ansible-playbook -i inventory-dns bind.yml --limit eastcoast -k
```
### Slave - ns2.mydomain.com
```bash
ansible-playbook -i inventory-dns bind.yml --limit westcoast -k
```

# Code Testing
It is assumed that all development or testing work are to be done in Windows 10 using WSL2 with Ubuntu.

## Environment Setup 
### Update OS
```bash
sudo apt update
sudo apt upgrade
```
### Install Packages
```bash
sudo apt install python
sudo rm -f /usr/bin/python
sudo ln -s /usr/bin/python3 /usr/bin/python
sudo apt install python3-pip
sudo apt install make
sudo pip3 install packaging
```
### Install Ansible
```bash
wget https://releases.ansible.com/ansible/ansible-latest.tar.gz
tar xvfz ansible-latest.tar.gz
cd ansible-<version>
sudo pip3 install -r requirements.txt
make
sudo make install
```
### Install Ansible Lint
```bash
sudo pip3 install ansible-lint
```
### Install Molecule
```bash
sudo pip3 install molecule
sudo pip3 install molecule[docker]
```
### Install Docker
```bash
sudo apt remove docker docker-engine docker.io containerd runc
sudo apt install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io
sudo apt upgrade
```
### Test Docker Install
```bash
sudo service docker start
sudo docker run hello-world
```
### Fix SystemD Issue
WSL2 doesn't properly support SystemD per following:
* https://github.com/microsoft/WSL/issues/994
* https://github.com/systemd/systemd/issues/8036

Various work arounds are available:
* https://github.com/arkane-systems/genie
* https://github.com/shayne/wsl2-hacks
* https://github.com/DamionGans/ubuntu-wsl2-systemd-script
* https://superuser.com/questions/1556609/how-to-enable-systemd-on-wsl2-ubuntu-20-and-centos-8

We will use genie and the following install guide is more comprehensive than the official page.
* https://gist.github.com/tdcosta100/385636cbae39fc8cd0937139e87b1c74
```bash
sudo genie -v -i
sudo genie -v -s
```
### Connect Docker Bridge Network To Outside
This is assuming the physical NIC to the Internet is eth0.  If unsure use 'ifconfig' to check.
```bash
apt-get install bridge-utils
apt-get install net-tools
brctl addif docker0 eth0
```
## Run Molecule
Following assumes we are within the genie bottle.  Otherwise sudo is required when running molecule.
### Setup Environment
Assume Windows 10 login is 'joe' and project name is ansible-bind, and IntelliJ is the default IDE.
```bash
USER_NAME=joe
ROLE_NAME=bind
PROJECT_NAME=ansible-bind
ROLE_PATH="/mnt/c/Users/${USER_NAME}/IdeaProjects/${PROJECT_NAME}/roles/${ROLE_NAME}"
MOLECULE_PATH="/mnt/c/Users/${USER_NAME}/IdeaProjects/${PROJECT_NAME}/roles/${ROLE_NAME}/molecule"
MOLECULE_ROLE_PATH="${MOLECULE_PATH}/default/roles/test_${ROLE_NAME}"
```
### Create Docker Instance
Note it is necessary to run ‘molecule  destroy’ first if Molecule has created containers in the past, even if the containers have been removed using Docker commands.
```bash
cd ${ROLE_PATH}
molecule create
```
### Check Docker Instance(s)
```bash
docker ps
```
### Run Playbook Being Tested
Note the two tags being skipped as they aren't supported within the docker images.
```bash
molecule converge -- --skip-tags sethostname,permit-port53
```
### Lint Playbooks
```bash
molecule lint
```
### Run Tests on Container State
```bash
molecule verify
```
### Cleanup
```bash
molecule destroy
```

# License
Apache License 2.0
