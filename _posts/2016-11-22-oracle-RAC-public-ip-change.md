---
layout: post
title: Oracle RAC change public IP addresses
published: false
---

All the IP's are being changed at my work and so I had to reconfigure a RAC to accept the VIP and the scan changes. Both changes are quite easy at least if you find the good documents on metalink. There are some blogs also but they are a bit outdated

## Collect info

get the clustername, the scan name, the nodes the old and the new IP's 

clustername: mycluster-s-cl (see metalink note 577300.1 on how to get it, or simply use olsnodes -c)

database name: mydb (use srvctl config)

nodes
```
#olsnodes -i
```

old and new IP's

node 1

eth0

172.23.14.50/255.255.0.0 ---> will become 10.143.12.21/255.255.248.0 (gateway 10.143.8.1)

vip

172.23.14.52/255.255.0.0 (check with  srvctl config nodeapps) ---> will become 10.143.12.23/255.255.248.0

node 2

eth0

172.23.14.51/255.255.0.0 ---> will become 10.143.12.22/255.255.240.0 (gateway 10.143.8.1)

vip

172.23.14.53/255.255.0.0 ---> will become 10.143.12.24/255.255.248.0

scan

Scan name: myscan-name (get it with srvctl config scan)

IP: 172.23.14.54, 172.23.14.55, 172.23.14.56 (netmask 255.255.0.0) will become 10.143.12.25/6/7 netmask 10.143.8.1 

note: use IP calculator (online web site) to see the network and the netmask of your new IP, it is handy.

## Perform changes

as oracle

```
srvctl stop database -db mydb
```

as root
```
#oifcfg delif -global eth0
#oifcfg setif -global eth0/10.143.8.0/21:public
```

now let us change the vip

as grid
```
srvctl stop vip -n uefa-s-ora01 ---> it complains about the listener so let us use the -f option
srvctl stop vip -n uefa-s-ora01 -f
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

as root

```
srvctl modify nodeapps -n node1  -A 10.143.12.23/255.255.248.0/eth0

```

check
```
#srvctl config nodeapps -n node1 -a
```

on node2

```
#srvctl stop vip -n node2 -f
```
check with crsctl status resource and with ipconfig that it is down

change hosts file and DNS

change cluster config as root
```
#srvctl modify nodeapps -n uefa-s-ora01 -A 10.143.12.23/255.255.248.0/eth0
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

