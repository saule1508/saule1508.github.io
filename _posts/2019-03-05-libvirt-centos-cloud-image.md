---
layout: post
title: provision centos 7 guest on libvirt
published: false
---

I am making my lab at home based on KVM (I have a Fedora linux host) and I need a quick way to provision a Centos VM. Until now I was using virt-manager (GUI) to create a VM, attach the Centos DVD, boot it and go through the installer. It is ok but it is not possible to automate and furthermore there is a faster way: we can download a cloud image, boot a VM based on it and very quickly we'll have our new guest centos VM. No PXE boot/kickstart needed.

The main steps are to download a cloud image of Centos 7 (it is a VM in the familiar qcow format, i.e the qemu copy-on-write format), and then use a tool that allow to make a small bootable iso which will inject some info in the image during the first boot.

documentation:
* https://cloudinit.readthedocs.io/en/latest/topics/examples.html
* https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_atomic_host/7/html/installation_and_configuration_guide/setting_up_cloud_init
* https://packetpushers.net/cloud-init-demystified/ (not up-to-date but explains the concept well)
* https://www.cyberciti.biz/faq/create-vm-using-the-qcow2-image-file-in-kvm/

NB: I am not sure what requires root access on the host and what not. I use my non-root user (this use is in the group libvirt) but sometimes I have to sudo. Of course a pre-requesiste is to have libvirt installed and running.

1. Download the cloud image. 

This is fast because the image is small (895M).

```bash
# create a directory to store the downloaded image, mine is /data1/downloads
cd /data1/downloads 
wget https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2
```

get info about the size (file size and virtual size): the virtual size is 8.0G which is not a lot
```bash
qemu-img info CentOS-7-x86_64-GenericCloud.qcow2
```

resize the image (the space will not be allocated until it is not used)

```
qemu-img resize CentOS-7-x86_64-GenericCloud.qcow2 50G
```
 If we look again the info, the virtual size is 50G but the file size did not change (895M)

create a libvirt storage pool for the new guest, of course adapt the directory to your system

```bash
sudo mkdir /data2/virtpool/pg01
sudo chown pierre:pierre /data2/virtpool/pg01
virsh pool-create-as --name pg01 --type dir --target /data2/virtpool/pg01
```

we need to create two files called user-data and meta-data, :those two files will be put in the iso

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
# useful so that we can logon from the console should ssh not be available
chpasswd:
  list: |
     root:password
  expire: False  
network-interfaces: |
  iface eth0 inet static
  address 192.168.122.10
  network 192.168.122.0
  netmask 255.255.255.0
  broadcast 192.168.122.255
  gateway 192.168.122.1
bootcmd:
  - ifdown eth0
  - ifup eth0
EOF
```
Now with cloud-utils we can create a bootable iso. The help of this command says "Create a disk for cloud-init to utilize nocloud". At the end it uses user-data and meta-data to produce an iso with which we can boot our cloud image.

```bash
sudo yum install cloud-utils mkisofs genisoimage
cloud-localds pg01.iso user-data meta-data
```

With this iso and the cloud image we downloaded, we will be able to provision our new guest. 

Copy the downloaded cloud image in the pool. I do that using qemu-img convert but a simple copy is OK
```bash
qemu-img convert -O qcow2 /data1/downloads/CentOS-7-x86_64-GenericCloud.qcow2 /data2/virtpool/pg01/pg01.qcow2
```

Now we can create the VM. Note that this time I do it with sudo, otherwise I have an error with network not found. And indeed, if I do virsh net-list I don't see the default network...

```bash
sudo virt-install --name pg01 --memory 1024 --vcpus 2 --disk /data2/virtpool/pg01/pg01.qcow2,device=disk,bus=virtio --os-type generic --os-variant centos7.0 --virt-type kvm --network network=default,model=virtio --cdrom /data2/virtpool/pg01/pg01.iso 
```

The program tryes to start virt-viewer so that one can see the progress of the installation but it fails. We can simply wait a bit or get into virt-manager to see the console.

To get the IP of the new guest

```bash
virsh domifaddr pg01
```

then I can ssh with user ansible (that I specified in the user-data file) or with user centos (defined in the cloud image). Since I copied my publick key (via user-data file) I don't need a password. I strongly recommand to change the root password so that you can connect via the console later on if you are stuck with ssh (wrong network config for example)

```bash
ssh ansible@192.168.122.71
The authenticity of host '192.168.122.71 (192.168.122.71)' can't be established.
ECDSA key fingerprint is SHA256:c4D596+Qqf7JER1T5f8RqlcdkmG4o01Q4q3oIhG5u2M.
ECDSA key fingerprint is MD5:75:09:7c:db:72:a2:c1:7e:e6:99:e9:36:8b:94:2a:2e.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.122.71' (ECDSA) to the list of known hosts.
[ansible@pg01 ~]$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        50G  895M   50G   2% /
devtmpfs        897M     0  897M   0% /dev
tmpfs           919M     0  919M   0% /dev/shm
tmpfs           919M   17M  903M   2% /run
tmpfs           919M     0  919M   0% /sys/fs/cgroup
tmpfs           184M     0  184M   0% /run/user/1000
sudo su -
passwd
```

We can now eject the cdrom. 

```bash
virsh change-media pg01  hda --eject --config
```
The config option is to make the change permanent. The path of the device, hda, can be found with the command domblkinfo

```
virsh domblkinfo pg01 --all
```

We can get rid of the iso and user-data and meta-data
```
rm /data2/virtpool/pg01/pg01.iso /data2/virtpool/pg01/user-data /data2/virtpool/pg01/meta-data
```

What's left is:
* configure the VM to have a static IP
* Add a disk and create a volume group

## Static IP

What I would do 
1. change the dhcp range of the default network (NB: the default network uses NAT, if you need to access the VM's from outside the host then you must create a bridge network)

```bash
virsh net-edit default
```
and change the dhcp range 
```
    <dhcp>
      <range start='192.168.122.100' end='192.168.122.254'/>
    </dhcp>
```
so that IP's between 192.168.122.2 and 192.168.122.99 are reserved for static allocation.

We can see that the change is not active
```
virsh net-dumpxml default 
virsh net-dumpxml default --inactive
```
We must stop and restart the default network
```
virsh net-destroy default
virsh net-start default
virsh net-dumpxml default 
```
then we must restart libvirtd
```bash
systemctl restart libvirtd
```
and we must reboot our VM then wait that the new IP is allocated
```
virsh reboot pg01
```

2. Change the network configuration of the VM.

I believe this can be done via cloud-syspreps or via cloud-init, for the time being I will just do it manually...

get into the VM as root, and edit the file /etc/sysconfig/network-scripts/ifcfg-eth0

```
BOOTPROTO=static
NM_CONTROLLED="no"
DEVICE=eth0
HWADDR=< keep existing one >
IPADDR=192.168.122.10
PREFIX=24
GATEWAY=192.168.122.1
DNS1=192.168.122.1
DOMAIN=localnet
ONBOOT=yes
TYPE=Ethernet
USERCTL=no
```


3. Change the host name on the host

With libvird on the host, there is dnsmask server automatically started so that the content of the /etc/hosts on the host will be made available to the guests via DNS. This is great for vm's because VM's can connect to each other via DNS.

In /etc/hosts on the host 
```
192.168.122.10 pg01.localnet
```

## LVM

In the my VM I always add a second disk and make a volume group + various file systems for my databases, soft,... since I have a background of Oracle DBA, I like to keep the Oracle convention to make file systems called /u01, /u02 and /u03, .... in my case I will have enough with /u01

1. Add a disk to the VM




