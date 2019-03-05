---
layout: post
title: provision centos 7 guest on libvirt
published: false
---

if you want a centos guest on libvirt but you don't want to spend time in the anaconda installer then there is an easier way which can be fully automated.

documentation:
* https://cloudinit.readthedocs.io/en/latest/topics/examples.html
* https://www.cyberciti.biz/faq/create-vm-using-the-qcow2-image-file-in-kvm/

```bash
sudo mkdir /u02/download
sudo chown oracle:oinstall /u02/downlowd
cd /u02/download
wget https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2
```
resize the image
```
qemu-img info CentOS-7-x86_64-GenericCloud.qcow2
qemu-img resize CentOS-7-x86_64-GenericCloud.qcow2 50G
```

create a storage pool for the new guest

```bash
sudo mkdir /u01/virtpool/pg01
virsh pool-create-as --name pg01 --type dir --target /u02/virtpool/pg01
```

we need to create two meta-data files, user-data and meta-data, those two files will be put in the iso

```bash
sudo su -
cat > /u02/virtpool/pg01/meta-data <<EOF
instance-id: pg01
local-hostname: pg01
EOF
```

```bash
sudo su -
cat > /u02/virtpool/pg01/user-data <<EOF
#cloud-config
# Hostname management
preserve_hostname: False
hostname: pg01
fqdn: pg01.localnet
# Setup Users with ssh keys so that I can log in into new machine
users:
  - default
  - name: ansible
    groups: ['wheel']
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh-authorized-keys:
      - $(cat /home/oracle/.ssh/id_rsa.pub)
# set timezone for VM
timezone: Europe/Brussels
# Remove cloud-init when finished with it
runcmd:
  - [ yum, -y, remove, cloud-init ]
# Configure where output will go
output: 
  all: ">> /var/log/cloud-init.log"
# configure interaction with ssh server
ssh_deletekeys: True
ssh_genkeytypes: ['rsa', 'ecdsa']
# Install my public ssh key to the first user-defined user configured 
# in cloud.cfg in the template (which is centos for CentOS cloud images)
ssh_authorized_keys:
  - ssh-rsa $(cat /home/oracle/.ssh/id_rsa.pub)
EOF
```

```bash
sudo qemu-img convert -O qcow2 /u02/download/CentOS-7-x86_64-GenericCloud.qcow2 /u02/virtpool/pg01/pg01.img
```

```bash
sudo yum install cloud-utils
sudo cloud-localds pg01.iso user-data meta-data

```

```bash
sudo virt-install --name pg01 --memory 1024 --vcpus 2 --disk /u02/virtpool/pg01/pg01.img,device=disk,bus=virtio --disk /u02/virtpool/pg01/pg01.iso,device=cdrom --os-type generic --os-variant centos7.0 --virt-type kvm --network network=default,model=virtio 

```

```bash
virsh change-media pg01  hda --eject --config
sudo rm /u02/virtpool/pg01/pg01.iso
```
