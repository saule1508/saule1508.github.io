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

NB: Normally one can do everything with a non-root user as long that the user is in the group libvirt and the env variable LIBVIRT_DEFAULT_URI is set (see below).

## Install KVM and libvirt

libvirt is normally installed by default on Centos 7, if not https://www.linuxtechi.com/install-kvm-hypervisor-on-centos-7-and-rhel-7/

You don't need to become root to create guest virtual machines, you can keep your own user but then do the following

* add user to group libvirt

In my case I am using user pierre, so:
```
sudo usermod -G libvirt -a pierre
```

* add environment variable LIBVIRT_DEFAULT_URI

```bash
export LIBVIRT_DEFAULT_URI=qemu:///system
```
and add it to your profile also
```bash
echo export LIBVIRT_DEFAULT_URI=qemu:///system >> ~/.bash_profile
```

## Download the cloud image. 

This is fast because the image is small (895M).

```bash
# create a directory to store the downloaded image, mine is /data1/downloads
cd /data1/downloads 
wget https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2
```

get info about the size (file size and virtual size)
```bash
qemu-img info CentOS-7-x86_64-GenericCloud.qcow2
```
As you can see the image the virtual size is 8.0G
```
image: CentOS-7-x86_64-GenericCloud.qcow2
file format: qcow2
virtual size: 8.0G (8589934592 bytes)
disk size: 895M
cluster_size: 65536
Format specific information:
    compat: 0.10
```
I prefer to resize the image (the space will not be allocated until it is used) so that I get a bigger / partition

```
qemu-img resize CentOS-7-x86_64-GenericCloud.qcow2 50G
```
 If we look again the info, the virtual size is 50G but the file size did not change (895M)

create a libvirt storage pool for the new guest, of course adapt the directory to your system. I use an env variable to point to the directory

```bash
export VMPOOLDIR=/data2/virtpool/pg01
sudo mkdir $VMPOOLDIR
sudo chown pierre:pierre $VMPOOLDIR
virsh pool-create-as --name pg01 --type dir --target $$VMPOOLDIR
```

## Prepare the iso

we need to create two files called user-data and meta-data, those two files will be put in the iso. 

Since I like to have static IP for my guest, I included the section network-interfaces in the meta-data file. If you don't need a fixed IP (or prefer to configure it yourself after), remove this section and a few line from the file user-data as well (see below) so that an IP will be assigned via dhcp by libvirt. 

```bash
cat > $VMPOOLDIR/meta-data <<EOF
instance-id: pg01
local-hostname: pg01
network-interfaces: |
  iface eth0 inet static
  address 192.168.122.10
  network 192.168.122.0
  netmask 255.255.255.0
  broadcast 192.168.122.255
  gateway 192.168.122.1
EOF
```
Before doing the next step, make sure you have a ssh keys pair in your home directory (subdirectory .ssh). If not the case generate the key pair now

```
ssh-keygen -t rsa
```
the public key will he injected in the cloud image, so that you will be able to log on from the host via ssh. 

To have the static IP configured, I added some hacking in the runcmd section below. Maybe there is a way to express that I want NM_CONTROLLED to no and ONBOOT to yes via the network section in the meta-data file, but it did not work. The commands ifdown/ifup comes from red-hat documentation (workaround for a bug ?). So if you don't need a static IP, remove the network-interfaces section from above and remove all lines from section runcmd below (except the first, i.e. keep the remove of cloud-init), remove also the DNS and resolv_conf settings below.

Of course adapt the hostname to your need...

See the chpasswd section, to set a password for root. This is super handy in case of problems because you can log through the console. But for some reason the chpasswd section below does not seem to work anymore (it did work at some point..)

```bash
cat > $VMPOOLDIR/user-data <<EOF
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
# some patches of network config that I could not do via network section in meta-data
runcmd:
  - [ yum, -y, remove, cloud-init ]
  - [ ifdown, eth0 ]
  - [ ifup, eth0 ]
  - [ sed, -i, -e, "s/ONBOOT=no/ONBOOT=yes/", /etc/sysconfig/network-scripts/ifcfg-eth0 ]
  - [ sed, -i, -e, "\$aNM_CONTROLLED=no", /etc/sysconfig/network-scripts/ifcfg-eth0 ]
# Set DNS
manage_resolv_conf: true
# we'll use dnsmask on the host
resolv_conf:
  nameservers: ['192.168.122.1']

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
EOF
```
Now with cloud-utils we can create a bootable iso. The help of this command says "Create a disk for cloud-init to utilize nocloud". At the end it uses user-data and meta-data to produce an iso with which we can boot our cloud image.

```bash
sudo yum install cloud-utils mkisofs genisoimage
cd $VMPOOLDIR
cloud-localds pg01.iso user-data meta-data
```

With this iso and the cloud image we downloaded, we will be able to provision our new guest. 

## create a new guest

First copy the downloaded cloud image in the pool. I do that using qemu-img convert but a simple copy is OK
```bash
qemu-img convert -O qcow2 /data1/downloads/CentOS-7-x86_64-GenericCloud.qcow2 $VMPOOLDIR/pg01.qcow2
```

then run virt-install to create the VM

```bash
virt-install --name pg01 --memory 1024 --vcpus 2 --disk $VMPOOLDIR/pg01.qcow2,device=disk,bus=virtio --os-type generic --os-variant centos7.0 --virt-type kvm --network network=default,model=virtio --cdrom $VMPOOLDIR/pg01.iso 
```

If you do not specify --noautoconsole in the virt-install command, the program tries to start virt-viewer so that one can see the progress of the installation. When at the end, it prompts for a login, reboot the VM. If you have --noautoconsole then just wait long enough (a few minutes, it is very fast)

If you specified a static IP, then you know the IP otherwise you can get the IP of the new guest:

```bash
virsh domifaddr pg01
```

then I can ssh with user ansible (that I specified in the user-data file) or with user centos (defined in the cloud image). Since I copied my public key (via user-data file) I don't need a password. I strongly recommand to change the root password so that you can connect via the console later on if you are stuck with ssh (wrong network config for example)

We can get rid of the iso and user-data and meta-data
```
rm $VMPOOLDIR/pg01.iso $VMPOOLDIR/user-data $VMPOOLDIR/meta-data
```

What's left is:
* configure the VM to have a static IP (if not done via cloud-config)
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

If doing it via cloud-init did not work, I would just do it manually...

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


3. add vm in /etc/hosts on the host

With libvird on the host, there is dnsmask server automatically started so that the content of the /etc/hosts on the host will be made available to the guests via DNS. This is great for vm's because VM's can connect to each other via DNS.

In /etc/hosts on the host 
```
192.168.122.10 pg01.localnet
```

## LVM 

first create a disk on the host

```bash
virsh vol-create-as --pool pg01 --name pg01-disk1.qcow2 --capacity 40G --allocation 10G --format qcow2
```
and allocate it to the VM
```bash
virsh attach-disk --domain pg01 --source /u01/virt/pg01/pg01-disk1.qcow2 --target vdb --persistent --driver qemu --subdriver qcow2
```
Then get into the VM and set-up the volume group, the logical volume and the file system
```bash
# as user root
yum install lvm2
pvcreate /dev/vdb
vgcreate vg01 /dev/vdb
lvcreate -l100%FREE -n lv_01 vg01
mkfs -t xfs /dev/vg01/lv_01
mkdir /u01
```

edit /etc/fstab and add
```
/dev/vg01/lv_01 /u01 xfs defaults 0 0
```

then mount the filesystem

```bash
mount /u01
```

The VM can be cloned, you can take snapshot, ... you can do pretty everything via the command line. I'll document that in a next post.
