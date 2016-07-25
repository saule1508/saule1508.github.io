---
published: false
---
## upgrade oracle 12.1.0.1 to 12.1.0.2

I have a RAC production setup with a standby database (the standby is single-instance and non-ASM) and I must upgrade it from 12.1.0.1 to 12.1.0.2. I think this is a bit stupid because Oracle 12.2 is about to be released, but I don't have the choice to wait for 12.2.

In those case it is important to do a rehearsal and to be able to learn the process, that's why I have set-up a RAC on VMWare ESX. To mirror the production set-up I also added a dataguard (also on ESX). 

In the previous blog post I documented the grid infrastructure upgrade, which went smoothly. Now I'll document the database part.

The process is the following

- install the software on the standby (install soft only) and on the node 1 of the RAC (install soft only)
- run cluvfy
- run preupgrade scripts and do the fixup
- stop the apply on the standby, stop the standby, prepare the new home (copy spfile, password file, modifyl listener.ora, oratab etc.), then startup mount the standby with the new home. Restart the apply
- start dbua on the primary RAC instance 1

let's document those steps;

copy the files p17694377_121020_Linux-x86-64_1of8.zip and p17694377_121020_Linux-x86-64_2of8.zip to /u01/staging/ora12102 on evs-rv-orarac01 (node 1 of the primary) and on evs-rv-orarac03 (the standby, it is a single instance DB) and unzip the two files. Note: there are 8 files in total but the two first are enough.

on evs-rv-orarac01 (i.e primary), login as oracle, unset ORACLE_HOME, ORACLE_BASE, ORACLE_SID and start the installer. Choose the option "Install soft only" and choose Oracle RAC installation. In the pre-requesite I have an error with the swap and with a setting /etc/security/limits.conf which is missing, namely Maximum locked memory lock (PRV-0044) : I believe this is a bug in the installer because this setting is in fact defined in /etc/security/limits.d/99-grid-oracle-limits.conf.

Inside /etc/security/limits.d/99-grid-oracle-limits.conf, I have this

```
oracle soft nproc 2047
oracle hard nproc 16384
oracle soft nofile 1024
oracle hard nofile 65536
oracle soft stack 10240
oracle hard stack 32768
grid soft nproc 2047
grid hard nproc 16384
grid soft nofile 1024
grid hard nofile 65536
grid soft stack 10240
grid hard stack 32768

#following computed with huge_page_setting.sh (see above)

oracle soft memlock 2416640
oracle hard memlock 2416640

```

There is also a check failure with ntp because I did not have the -x option in /etc/sysconfig/ntp, so I added it

```
[root@evs-rv-orarac02 sysconfig]# cat /etc/sysconfig/ntpd
# Drop root to id 'ntp:ntp' by default.
OPTIONS="-u ntp:ntp -p /var/run/ntpd.pid -g -x"
```

Anyway, I choosed to ignore and I installed the soft on evs-rv-orarac01 (primary) and on evs-rv-orarac03 (standby).


**cluvfy**

This is a nice utility that must be downloaded from technet in the form of a zip file (cvupack_Linux_x86_64.zip) and unzipped in a directory owner by user oracle, in my case in /home/oracle/cvu
Note: before the grid upgrade, I downloaded, installed and run the same utility but for user grid. Since we are now upgrading the database we must run this script with the oracle user

```
 ./bin/cluvfy stage -pre dbinst -upgrade -src_dbhome /u01/app/oracle/product/12.1.0/dbhome_1 -dest_dbhome /u01/app/oracle/product/12.1.0/dbhome_2  -dest_version 12.1.0.2.0
```

**upgrade**

once all pre-check are ok, we can do the actual update. The idea is to start the standby from the new home and put it in apply state, so that when the primary is being upgraded it will apply the dictionnary changes done on the primary by the upgrade assistant.

on the standby - evs-rv-orarac03 -

stop the apply and stop the transport using dgmgrl

```
edit database 'DEVRACDB_MN' set state='TRANSPORT-OFF`;
edit database 'DEVRACDB_DG' set state='APPLY-OFF';
```

stop the database and stop the listener, then install the files in the new home

```
cd /u01/app/oracle/product/12.1.0
cp ./db_home1/dbs/spfileDEVRACDB3.ora ./db_home2/dbs/
cp ./db_home1/dbs/orapwDEVRACDB3 ./db_home2/dbs
cp ./db_home1/network/admin/*.ora ./db_home2/network/admin/
```

edit the listener.ora and change the oracle home
edit /etc/oratab and change the oracle home

switch to the new oracle home then start the database mount and start the listener. 

disable flash back

```
sqlplus / as sysdba
alter database set flashback off;
exit
```

Restart the transport and the apply using dgmgrl

```
dgmgrl
connect sys/...
edit database 'DEVRACDB_MN' set state='TRANSPORT-ON';
edit database 'DEVRACDB_DG' set state='APPLY-ON';
```

check the alert log of the dataguard

on the primary - evs-rv-orarac01 -

stop all dbms_scheduler jobs, comment out cron entries (i.e. backup)








Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.
