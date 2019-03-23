---
layout: post
title: Upgrade grid 12.1.0.1 to 12.1.0.2
published: true
---
I need to upgrade a RAC production set-up from grid 12.1.0.1 to 12.1.0.2. Since this is the first time I do that, I have first set-up a rehearsal environment on VMWare (to be documented in another post).
<!--more-->

The upgrade went very smoothly. It consists of:

- Creating the new home for Grid on all servers with the correct permissions (the old was /u01/app/12.0.1/grid and the new is /u01/app/12.0.1.2/grid);
- Run the cluvfy: this checks and eventually fixes issues pro-actively;
- Install the new soft on one of the server. During the install you can choose to have the upgrade taking place and so the nodes will be upgraded in a rolling fashion (and the new soft will be installed on both nodes).

There is a nice utility called cluvfy that must be downloaded from technet and installed in /home/grid (i.e. outside the ORACLE_HOME): this utility performs a lot of pre-check and generates fixup scripts.

So those are my two nodes

| node |
|------|
| evs-rv-orarac01 |
| evs-rv-orarac02 |

the grid home is /u01/app/12.1.0/grid. This is the content of /etc/oratab

```
+ASM1:/u01/app/12.1.0/grid:N:           # line added by Agent
-MGMTDB:/u01/app/12.1.0/grid:N:         # line added by Agent
DEVRACDB1:/u01/app/oracle/product/12.1.0/dbhome_1:N:
DEVRACDB_MN:/u01/app/oracle/product/12.1.0/dbhome_1:N:          # line added by Agent
```

On both nodes, create the new home

```bash
sudo mkdir -p /u01/app/12.1.0.2/grid
sudo chown root:oinstall /u01/app/12.1.0.2
sudo chown grid:oinstall /u01/app/12.1.0.2/grid
```

on node 1, unzip the soft in a staging directory

```bash
mkdir /u01/staging/grid12102 (owned by user grid)
cd /u01/staging/grid12102
unzip linuxamd64_12102_grid_1of2.zip 
unzip linuxamd64_12102_grid_2of2.zip
```

On node 1, run the cluvfy utility (downloaded from technet or metalink)

```bash
cd /home/grid/cvu
unzip cvupack_Linux_x86_64.zip
./bin/cluvfy stage -pre crsinst -upgrade -rolling -src_crshome /u01/app/12.1.0/grid -dest_crshome  /u01/app/12.1.0.2/grid -dest_version 12.1.0.2.0 -verbose
```

note that there is a similar utility in the grid directory when you unzip the soft. But the idea is that the cluvfy is more up to date. Anyway we can give it a run also. Change to the directory where it is located then

```bash
./runcluvfy.sh stage -pre crsinst -upgrade -src_crshome /u01/app/12.1.0/grid/ -dest_crshome /u01/app/12.1.0.2/grid -dest_version 12.1.0.2.0 -fixup -verbose
```

Get the current version (as user root)

```bash
crsctl query crs softwareversion -all
```

make sure there is no firewall blocking the communication on the interconnect

```bash
iptables -L
```

In case of doubts, remove all rules

```bash
iptables -F
```

Now we can do the install, as user grid on server 1 (where the soft files were unzipped)

```bash
unset ORACLE_SID
unset ORACLE_BASE
unset ORACLE_HOME

export PATH=/usr/local/bin:/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/sbin:/home/grid/bin
```

the last command remove the current ORACLE_HOME from the PATH

```bash
cd /u01/staging/grid12102/grid
./runInstaller
```

- choose the option : Upgrade Oracle grid infrastructure or Oracle ASM (it is selected by default)
- In software location, choose /u01/app/12.1.0.2/grid
- when prompted run the /u01/app/12.1.0.2/grid/orarootinstall.sh script on the first node and then on the second node (once finished on the first node). It takes time.

After the upgrade, check the active version, as root

```bash
crsctl query crs softwareversion -all
```

As you can see the oratab was also changed.
