---
layout: post
title: Oracle RAC change public IP addresses
published: true
---

Today I had to reconfigure a RAC because the public IP's are all changed (ip addresses of the hosts, ip addresses of the VIP and IP's of the scan. Both the VIP and the SCAN changes are quite easy at least if you find the good documents on metalink. There are some blogs also but they are a bit outdated
<!--more-->

## Collect info

get the clustername, the scan name, the nodes, the old and the new IP's and keep this information at hand. 

**clustername**: see metalink note 577300.1 on how to get it, or simply use olsnodes -c

**database name**: use srvctl config to get the name

**nodes names**: use olsnodes

```
#olsnodes -i
```

Then gather the old and new IP's for the public interface, in my case eth0, and keep them at hand

**node 1**

eth0

172.23.14.50/255.255.0.0 ---> will become 10.143.12.21/255.255.248.0 (gateway 10.143.8.1)

vip

172.23.14.52/255.255.0.0 (check with  srvctl config nodeapps) ---> will become 10.143.12.23/255.255.248.0

**node 2**

eth0

172.23.14.51/255.255.0.0 ---> will become 10.143.12.22/255.255.240.0 (gateway 10.143.8.1)

vip

172.23.14.53/255.255.0.0 ---> will become 10.143.12.24/255.255.248.0

**scan**

Scan name: get the scan name it with srvctl config scan

IP: 172.23.14.54, 172.23.14.55, 172.23.14.56 (netmask 255.255.0.0) will become 10.143.12.25/6/7 netmask 10.143.8.1 

note: use IP calculator (online web site) to see the network and the netmask of your new IP, it is handy.

## Perform changes

as oracle, I stopped the DB. However it is probably not needed.

```
srvctl stop database -db mydb
```

as root, delete the interface and recreate it with the new network

```
#oifcfg delif -global eth0
#oifcfg setif -global eth0/10.143.8.0/21:public
```

now it is time to change the vip

as user grid

```
srvctl stop vip -n <node name 1>
```

it complains about the listener so let us use the -f option

```
srvctl stop vip -n <node name 1> -f
```

check that the vip is down

```
crsctl status resource -t
```

check that the vip is not bound to eth0 anymore

```
ifconfig -a
```

Change /etc/hosts and change the DNS

Now, as root, we can modify the vip in the cluster config

```
#srvctl modify nodeapps -n <node name node1>  -A 10.143.12.23/255.255.248.0/eth0
```

check the result

```
#srvctl config nodeapps -n <node name 1> -a
```

on the second node, do the same

```
#srvctl stop vip -n <node name node2> -f
```

check with crsctl status resource and with ipconfig that it is down, then change hosts file and DNS and then change cluster config.

As user root

```
#srvctl modify nodeapps -n <node name node2> -A 10.143.12.24/255.255.248.0/eth0
```

## modify scan

it is easy and it is explained in doc 952903.1

first verify that the DNS resolves to the new IP's 

```
nslookup <scan-name>
```

as root, look the current config

```
srvctl config scan
```

```
srvctl stop scan_listener
srvctl stop scan
srvctl modify scan -n <scan name>
```

check the new config

```
srvctl config scan
```

## change IP at OS level

this implies changing /etc/sysconfig/network-scripts/ifcfg-eth0 and /etc/sysconfig/network, /etc/hosts, maybe the know_hosts in the home of oracle, grid and root.
