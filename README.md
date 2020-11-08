![Molecule Tests](https://github.com/kenrui-group/ansible-bind/workflows/Molecule%20Tests/badge.svg)

# Overview
Install or upgrade to latest BIND minor version and apply zone files.  This playbook assumes the server 
has already been set with the correct network details (eg IP, DNS servers, etc).

# Features
The playbook is idempotent and permits a regular rebuild of DNS servers to achieve the following:
* Install the latest [BIND](https://www.isc.org/bind/) available.
* Update of zone records.
* Rotate RNDC keys.
* Get / revoke certificate from [Let's Encrypt](https://letsencrypt.org/) using DNS challenge (wildcard certificate is supported).
* Check certificate expiry status by comparing server date time and certificate start / end date time.
* Test certificate issue / revoke using [Let's Encrypt Pebble](https://github.com/letsencrypt/pebble).
* Optional use of local LAN IPs or gateway IPs for zone transfer or refresh between master and slave (gateway / firewall unable to do hairpin NAT)
* Support inclusion of SPF and DKIM zone records.

Playbook also comes with settings for testing using Molecule.

# Configuration
Following variables are configurable in vars/main.yml.
* BIND major version to install or upgraded to.
* Domain name (eg mydomain.com).
* Hostnames and IP addresses for DNS A and PTR entries.
* Election to use local LAN IPs for DNS server and gateway for zone transfer and notify.
* SPF TXT record.
* DKIM selector and TXT record.

DNS servers' IPs to be managed via the playbooks are configured in the inventory-dns file.

---------------------------------------

Molecule is setup to assert changes done by the playbooks so configuration changes need to be synced to the following:
 - bind/molecule/default/roles/test_bind/vars/main.yml.
 - cert/molecule/default/roles/test_cert/vars/main.yml.

---------------------------------------

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
### Configuration
#### Playbook 'bind'
> roles/bind/vars/main.yml  
> roles/bind/molecule/default/roles/test_bind/vars/main.yml

Specify BIND version, domain, prefixes, and A records details.
```yaml
bindversion: "9.11"
domain: "mydomain.com"
ip_reverse: "0.17.172"
ip_reverse_zone_file_prefix: "172.17.0"
zone_records:
  uat:
    hostname: "uat"
    hostip: "4"
  www:
    hostname: "www"
    hostip: "5"
  ns1:
    hostname: "ns1"
    hostip: "2"
  ns2:
    hostname: "ns2"
    hostip: "3"
  confluence:
    hostname: "confluence"
    hostip: "6"
```

Specify `uselanip: true` to elect to use `lanip` values in environments where zone transfer, notify, refresh, etc appear to originate from private IPs rather than public IPs.   

```yaml
uselanip: true
zone_records:
  ns1:
    hostname: "ns1"
    hostip: "2"
    lanip: 10.0.1.10
  ns2:
    hostname: "ns2"
    hostip: "3"
    lanip: 10.0.2.10
```

Specify `usegatewayip: true` to elect to use `gatewayip` value in environments where zone transfer, notify, refresh, etc appear to originate from gateway IP rather than public IPs.  This may happen when DNS master notify slave on update or slave request from master via public IPs but firewall or gateway isn't able to perform hairpin NAT.
```yaml
usegatewayip: true
gatewayip: 192.168.50.1
```

Specify SPF and DKIM details.
```yaml
spf_value: "v=spf1 include:_spf.google.com ~all"
dkim_prefix_selector: "mail"
dkim_public_key: "v=DKIM1; k=rsa; p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAxNMFhiEv1MGpsDnBSKZb8I5WIlS4o88qtBKmkIMYaK5vSG1q+lhNzueFfLNAdPc4w/Srs1+CA+NacMin4QIMNRCgR3xeVrexE2o50ra4WEw5m74VjlmJbSTOF7wTDf66g1EBEuJ9kgLaCpVnzRuKSUefL/W5rxCTm+wT8xogZQJPcqN3VMmzZOdum5ruHjF5pEyk6t2VwBQJkTwlW9Ex1rhoPYFA7tzk1x7W+mUHoQemEOw34whEUg/jhUB712Vwtsk5DALYcz2bK6fD2sZQ5dXcD/mhnH/f91y/S5Os+7Xej+xXunpV5+V0bUdYhRC+7Zvoj8/T3t29VbIOgwz6yQIDAQAB"
```

#### Playbook 'cert'
> roles/cert/vars/main.yml  
> roles/cert/molecule/default/roles/test_cert/vars/main.yml

Specify organization details.
```yaml
domain: "mydomain.com"
organization_name: "MyDomain"
email_address: "support@mydomain.com"
common_name: "mydomain.com"
```

Prepend domain name with `*.` if wildcard certificate is to be generated.
```yaml
common_name: "*.mydomain.com"
```

### Run Playbooks
#### Playbook 'bind'
Run playbook against all DNS servers.
```bash
ansible-playbook -i inventory-dns bind.yml -k
```
Update eastcoast DNS servers only.
```bash
ansible-playbook -i inventory-dns bind.yml --limit eastcoast -k
```
Update westcoast DNS servers only.
```bash
ansible-playbook -i inventory-dns bind.yml --limit westcoast -k
```
#### Playbook 'cert'
Additional flags need to be used to instruct playbook on what to be done or which Let's Encrypt directory to use.
```bash
ansible-playbook -i inventory-dns cert.yml -e action=issue -e dir=staging -k
ansible-playbook -i inventory-dns cert.yml -e action=revoke -e dir=staging -k
ansible-playbook -i inventory-dns cert.yml -e action=check_account -e dir=staging -k
ansible-playbook -i inventory-dns cert.yml -e action=check_expiry -k
```


# Code Testing
Molecule testing is done via Github Actions and can run locally within WSL2 as well.

## Development Environment Setup
Development can be done on Windows 10 using WSL2 with Ubuntu with the following setup.
 
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
ROLE_NAME=bind # Check bind playbook
ROLE_NAME=cert # Check cert playbook
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

When testing 'cert' playbook additional flags need to be used to instruct playbook on what to be done.
Note the desired Let's Encrypt directory to use needs to be specified as one of 'pebble', 'staging', 'production'.
```bash
molecule converge -- --skip-tags sethostname,permit-port53 -e action=issue -e dir=pebble
molecule converge -- --skip-tags sethostname,permit-port53 -e action=revoke -e dir=pebble
molecule converge -- --skip-tags sethostname,permit-port53 -e action=check_account -e dir=pebble
molecule converge -- --skip-tags sethostname,permit-port53 -e action=check_expiry
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
