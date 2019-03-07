---
layout: post
title: provision centos 7 guest on libvirt
published: false
---

Since I am making my lab based on KVM (I have a linux host), I wanted a quick way to provision a Centos VM. Until now I was using virt-manager (GUI) to create a VM, attach the Centos DVD, boot it and go through the installer. It is ok but there is a faster and an easier to automate way.

The main steps are to download a cloud image of Centos 7 (it is a VM in the familiar qcow format, i.e the qemu copy-on-write format), and then use a tool that allow to make a small bootable iso which will inject some info in the image during the first boot.

documentation:
* https://cloudinit.readthedocs.io/en/latest/topics/examples.html
* https://www.cyberciti.biz/faq/create-vm-using-the-qcow2-image-file-in-kvm/

I am not sure what requires root access on the host and what not. I try to use my personal user but sometimes I have to sudo. On my host I am using a user pierre (the user is in the group libvirt). Of course a pre-requesiste is to have libvirt installed and running.

1. Download the cloud image. 

This is fast because the image is small (895M).

```bash
# create a directory to store the downloaded image, mine is /data1/downloads
cd /data1/downloads 
wget https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2
```

resize the image (the space will not be allocated until it is not used)

```
qemu-img info CentOS-7-x86_64-GenericCloud.qcow2
# this shows that the disk size is 895M and the virtual size is 8.0 G, which is not a lot
qemu-img resize CentOS-7-x86_64-GenericCloud.qcow2 50G
# qemu-img info shows now that the virtual size is 50G
# 
```
create a libvirt storage pool for the new guest, of course adapt the directory to your system

```bash
sudo mkdir /data2/virtpool/pg01
sudo chown pierre:pierre /data2/virtpool/pg01
virsh pool-create-as --name pg01 --type dir --target /data2/virtpool/pg01
```

we need to create two meta-data files, user-data and meta-data, those two files will be put in the iso

```bash
cat > /data2/virtpool/pg01/meta-data <<EOF
instance-id: pg01
local-hostname: pg01
EOF
```
Before doing the next step, make sure you have a ssh keys pair in your home directory (subdirectory .ssh). If not the case generate the key pair now

```
ssh-keygen -t rsa
```
the public key will he injected in the cloud image, so that you will be able to log on from the host via ssh. 

```bash
cat > /data2/virtpool/pg01/user-data <<EOF
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
      - $(cat $HOME/.ssh/id_rsa.pub)
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
  - ssh-rsa $(cat $HOME/.ssh/id_rsa.pub)
EOF
```
Copy the downloaded cloud image in the pool. I do that using qemu-img convert but a simple copy would work
```bash
qemu-img convert -O qcow2 /data1/downloads/CentOS-7-x86_64-GenericCloud.qcow2 /data2/virtpool/pg01/pg01.qcow2
```
Now with cloud-utils we can create a bootable iso. The help of this command says "Create a disk for cloud-init to utilize nocloud". At the end it uses user-data and meta-data to produce an iso with which we can boot our cloud image.

```bash
sudo yum install cloud-utils mkisofs genisoimage
sudo cloud-localds pg01.iso user-data meta-data

```
Now we can create the VM. Note that this time I do it with sudo, otherwise I have an error with network not found. And indeed, if I do virsh net-list I don't see the default network...

```bash
sudo virt-install --name pg01 --memory 1024 --vcpus 2 --disk /data2/virtpool/pg01/pg01.qcow2,device=disk,bus=virtio --disk /data2/virtpool/pg01/pg01.iso,device=cdrom --os-type generic --os-variant centos7.0 --virt-type kvm --network network=default,model=virtio --cdrom /data2/virtpool/pg01/pg01.iso 
```

```bash
virsh change-media pg01  hda --eject --config
rm /data2/virtpool/pg01/pg01.iso
```
