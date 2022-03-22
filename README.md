# vmware-openstack-rating

# global.rating.yml is for core services + rating services 

# create environment
 
sudo dnf install python3-pip
sudo dnf install python3-devel libffi-devel gcc openssl-devel python3-libselinux
python3 -m venv /venv-1
python3.8 -m venv /venv-2
source /venv-1/bin/activate
source /venv-2/bin/activate
pip install --upgrade pip
pip install ansible
pip install pyvmomi                                                                                                                                                
pip install git+https://opendev.org/openstack/kolla-ansible@master
pip install git+https://opendev.org/openstack/kolla-ansible@stable/xena

## pip list to show version
Python              3.6.8
pip                 21.3.1
ansible             4.10.0
kolla-ansible       13.1.0.dev281
pyvmomi             7.0.3



## pip list to show version
pip install -U 'ansible<3.0'

Python             3.8.12
pip                22.0.4


pyvmomi            7.0.3



cd /venv-1/share
git clone https://github.com/chuhakhanh/vmware-openstack-rating

# deploy openstack base centos-stream-8
## create VM 
ansible-playbook deploy_vmware_rating_cluster.yml 
## prepare openstack environment
for i in control-1 control-2 control-3;
do 
  sshpass -p alo1234 ssh-copy-id -f -i ~/.ssh/id_rsa.pub root@$i ; 
done
## prepare openstack
ansible-playbook -i all-in-one-control-1 prepare_all_node.yml
ansible-playbook -i all-in-one-control-1 prepare_storage_lvm.yml

ansible-playbook -i all-in-one-control-2 prepare_all_node.yml
ansible-playbook -i all-in-one-control-2 prepare_storage_lvm.yml

ansible-playbook -i all-in-one-control-3 prepare_all_node.yml
ansible-playbook -i all-in-one-control-3 prepare_storage_lvm.yml

## prepare kolla config file 

cp ./kolla/globals.centos8.rating.yml ./kolla/globals.yml
cp ./kolla/globals.centos8.telemetry.yml ./kolla/globals.yml
cat ./kolla/globals.yml | grep . | grep -v "#"

kolla-ansible --configdir ./kolla -i ./all-in-one-control-1 bootstrap-servers
kolla-ansible --configdir ./kolla -i ./all-in-one-control-1 prechecks
kolla-ansible --configdir ./kolla -i ./all-in-one-control-1 deploy

## kolla_internal_vip_address: "10.1.17.52"
kolla-ansible --configdir ./kolla -i ./all-in-one-control-2 bootstrap-servers
kolla-ansible --configdir ./kolla -i ./all-in-one-control-2 prechecks
kolla-ansible --configdir ./kolla -i ./all-in-one-control-2 deploy
## kolla_internal_vip_address: "10.1.17.53"
kolla-ansible --configdir ./kolla -i ./all-in-one-control-3 bootstrap-servers
kolla-ansible --configdir ./kolla -i ./all-in-one-control-3 prechecks
kolla-ansible --configdir ./kolla -i ./all-in-one-control-3 deploy

# configure openstack
. /etc/kolla/admin-openrc-c1.sh
openstack image create "cirros" --file /root/cirros-0.4.0-x86_64-disk.img --disk-format qcow2 --container-format bare --public
openstack flavor create --id 1 --ram 1024 --disk 1  --vcpu 1 tiny
openstack flavor create --id 2 --ram 4096 --disk 10 --vcpu 2 small
openstack flavor create --id 4 --ram 8096 --disk 50 --vcpu 2 medium
openstack network create --share --provider-physical-network physnet1 --provider-network-type vlan --provider-segment=111 pro-vlan111
openstack subnet create --subnet-range 10.1.0.0/16 --gateway 10.1.0.1 --network pro-vlan111 --allocation-pool start=10.1.17.130,end=10.1.17.150 pro-vlan111-subnet1
openstack network create --share --provider-physical-network physnet1 --provider-network-type vlan --provider-segment=126 pro-vlan126
openstack subnet create --subnet-range 192.168.126.0/24 --gateway 192.168.126.1 --network pro-vlan126 --allocation-pool start=192.168.126.130,end=192.168.126.150 pro-vlan126-subnet1
net-id-pro-vlan111='openstack network list | grep pro-vlan111 | cut -f2 -d"|"'
openstack server create --flavor 1 --image cirros --nic net-id=0507983d-b3a1-40b0-83c8-9f0cf4f4f6da inst1
openstack server create --flavor 1 --image cirros --nic net-id=$net-id-pro-vlan111 inst1


