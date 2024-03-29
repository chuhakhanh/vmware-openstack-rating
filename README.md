# vmware-openstack-rating
## references:
Cloudkitty metrics:
https://docs.openstack.org/cloudkitty/latest/admin/configuration/collector.html#minimal-configuration

Python3:
https://computingforgeeks.com/how-to-install-python-3-on-centos/

Ansible Openstack:
https://docs.ansible.com/ansible/latest/collections/openstack/cloud/index.html

# global.rating.yml is for core services + rating services 
# all-in-one in control-2

# create environment
 
sudo dnf install python3-pip
sudo dnf install python3-devel libffi-devel gcc openssl-devel python3-libselinux
pip install git+https://opendev.org/openstack/kolla-ansible@stable/xena
pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/xena

python3.8 -m venv /venv-2
source /venv-2/bin/activate
pip install --upgrade pip
pip install -U 'ansible<3.0'
pip install pyvmomi                                                                                                                                                

## pip list to show version
pip3 install 'ansible==2.10.7'

pip list
ansible            2.10.7
ansible-base       2.10.17
kolla-ansible      13.0.2.dev92
pip                22.0.4
pyvmomi            7.0.3
openstacksdk           0.59.0

cd /venv-2/share
git clone https://github.com/chuhakhanh/vmware-openstack-rating


# deploy openstack base centos-stream-8
## create VM 
ansible-playbook deploy_vmware_rating_cluster.yml 
## prepare openstack environment
for i in control-2 ;
do 
  sshpass -p alo1234 ssh-copy-id -f -i ~/.ssh/id_rsa.pub root@$i ; 
done
## prepare openstack
ansible-playbook -i all-in-one-control-2 prepare_all_node.yml
ansible-playbook -i all-in-one-control-2 prepare_storage_lvm.yml

## prepare kolla config file 
### password file in the repo is pre-generated 
### or can be create  
cp /venv-2/share/kolla-ansible/etc_examples/kolla/passwords.yml /etc/kolla/
kolla-genpwd 
cp /etc/kolla/passwords.yml /venv-2/share/vmware-openstack-rating/kolla

### cp globals.yml for rating
cp ./kolla/globals.centos8.rating.yml ./kolla/globals.yml
cat ./kolla/globals.yml | grep . | grep -v "#"

## kolla_internal_vip_address: "10.1.17.52"
kolla-ansible --configdir ./kolla -i ./all-in-one-control-2 bootstrap-servers

  
    rm /var/run/libvirt/libvirt-sock
#### disable prechecks_enable_host_ntp_checks: false
    /venv-2/share/kolla-ansible/ansible/roles/prechecks/defaults/main.yml
    prechecks_enable_host_ntp_checks: false
 

 #### configure fix error local registry
    mv /etc/systemd/system/docker.service /etc/systemd/system/docker.service_save; systemctl daemon-reload; systemctl restart docker; systemctl status docker
    vi /etc/sysconfig/docker
    vi /etc/docker/daemon.json
    {
        "bridge": "none",
        "insecure-registries" : ["repo-1:4000"],
        "ip-forward": false,
        "iptables": false,
        "log-opts": {
            "max-file": "5",
            "max-size": "50m"
        }
    }
#### configure main.yml to use local repo docker    
    vi /venv-2/share/kolla-ansible/ansible/roles/baremetal/defaults/main.yml
    #docker_yum_url: "https://download.docker.com/linux/centos"
    docker_yum_url: http://repo-1/2022_03/docker/docker
    docker_yum_baseurl: "{{ docker_yum_url }}/$releasever/$basearch/stable"
    docker_yum_gpgkey: "{{ docker_yum_url }}/gpg"
    #docker_yum_gpgcheck: true
    docker_yum_gpgcheck: no

kolla-ansible --configdir ./kolla -i ./all-in-one-control-2 prechecks
kolla-ansible --configdir ./kolla -i ./all-in-one-control-2 deploy
kolla-ansible post-deploy 
source /etc/kolla/admin-openrc-c1.sh 

# deploy demo project in openstack
ansible-playbook -i all-in-one-control-2 deploy_demo_project.yml
ansible-playbook -i all-in-one-control-2 deploy_demo_project.yml --start-at-task="Create security group rules"


