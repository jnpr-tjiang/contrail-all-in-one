# Instructions for installing Contrail Micro Services with Kolla

# 1. Setup base host on Centos 7.4
```
yum -y install epel-release
yum -y remove  python-jinja2
yum -y install ansible-2.3.1.0
yum -y install centos-release-openstack-ocata
yum -y install python-oslo-config
yum -y install git
yum -y install python-pip
pip install --upgrade Jinja2
```

# 2. Get all-in-one ansible script
```
cd /root
git clone https://github.com/rrugge/contrail-all-in-one.git
cd /root/contrail-all-in-one
```

# 3. Edit inventory/group_vars/all.yml
Set the ip-addr for CONTROLLER_NODES
```
CONTROLLER_NODES: 10.84.24.1
NTP_SERVER: 10.84.5.100
CONTRAIL_VERSION: 5.0.0-134-centos7-ocata
CONTAINER_REGISTRY: michaelhenkel
```

# 4. Setup config in kolla and contrail

```
cd /root/contrail-all-in-one
ansible-playbook -i inventory/ playbooks/deploy.yml
```

# 5. Kolla bootstrap
```
cd /root/contrail-kolla-ansible/ansible
ansible-playbook -i inventory/all-in-one -e@../etc/kolla/globals.yml -e@../etc/kolla/passwords.yml -e action=bootstrap-servers kolla-host.yml
```

# 6. Contrail bootstrap [NOTE: set working NTP ip address]
```
cd /root/contrail-ansible-deployer
ansible-playbook -e '{"CONFIGURE_VMS":true}' -e '{"CONTAINER_VM_CONFIG":{"network":{"ntpserver":"10.84.5.100"}}}' -t configure_vms -i inventory/ playbooks/deploy.yml
```

# 7. Kolla deploy
```
cd /root/contrail-kolla-ansible/ansible
ansible-playbook -i inventory/all-in-one -e@../etc/kolla/globals.yml -e@../etc/kolla/passwords.yml -e action=deploy site.yml
```

# 8. Contrail deploy
```
cd /root/contrail-ansible-deployer
ansible-playbook -e '{"CREATE_CONTAINERS":true}' -i inventory/ playbooks/deploy.yml
```

# 9. Test your setup with VM to VM ping
```
source /etc/kolla/admin-openrc.sh
wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
openstack image create cirros2 --disk-format qcow2 --public --container-format bare --file cirros-0.4.0-x86_64-disk.img                                      
openstack network create testvn
openstack subnet create --subnet-range 192.168.100.0/24 --network testvn subnet1
openstack flavor create --ram 512 --disk 1 --vcpus 1 m1.tiny 
NET_ID=`openstack network list | grep testvn | awk -F '|' '{print $2}' | tr -d ' '`
openstack server create --flavor m1.tiny --image cirros2 --nic net-id=${NET_ID} test_vm1
openstack server create --flavor m1.tiny --image cirros2 --nic net-id=${NET_ID} test_vm2
```
