---
published: true
---
## upgrade oracle 12.1.0.1 to 12.1.0.2

I have a RAC production setup with a standby database (the standby is single-instance and non-ASM) and I must upgrade it from 12.1.0.1 to 12.1.0.2. 

In those case it is important to do a rehearsal and to be able to learn the process, that's why I have set-up a RAC on VMWare ESX. To mirror the production set-up I also added a dataguard (also on ESX). 

In the previous blog post I documented the grid infrastructure upgrade, which went smoothly. Now I'll document the database part.

The process is the following

- install the software on the standby (install soft only) and on the node 1 of the RAC (install soft only)
- run cluvfy
- run preupgrade scripts and do the fixup
- disable the broker configuration and set dg_broker_start to false scope=both 
- stop the standby, copy spfile, password file, listener.ora from old to new home and edit the listener.ora, oratab. Keep the standby shutdown
- start dbua on the primary RAC instance 1
- startup mount the standby and re-enable the broker configuration
- do the postfixup scripts

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

This is a nice utility that must be downloaded from technet in the form of a zip file (cvupack_Linux_x86_64.zip) and unzipped in a directory owned by user oracle, in my case in /home/oracle/cvu
Note: before the grid upgrade, I downloaded, installed and run the same utility but for user grid. Since we are now upgrading the database we must run this script with the oracle user

```
 ./bin/cluvfy stage -pre dbinst -upgrade -src_dbhome /u01/app/oracle/product/12.1.0/dbhome_1 -dest_dbhome /u01/app/oracle/product/12.1.0/dbhome_2  -dest_version 12.1.0.2.0
```

**pre-upgrade**

on the primary, copy preupgrd.sql and utluppkg.sql from the new home dbs/admin to a directory of your choice, in my case /home/oracle/upgrade12102. Then run preupgrd.sql as sysdba (from the old home). It will says something like 

```
ACTIONS REQUIRED:

1. Review results of the pre-upgrade checks:
 /u01/app/oracle/cfgtoollogs/DEVRACDB_MN/preupgrade/preupgrade.log

2. Execute in the SOURCE environment BEFORE upgrade:
 /u01/app/oracle/cfgtoollogs/DEVRACDB_MN/preupgrade/preupgrade_fixups.sql

3. Execute in the NEW environment AFTER upgrade:
 /u01/app/oracle/cfgtoollogs/DEVRACDB_MN/preupgrade/postupgrade_fixups.sql

```

and of course one must carefully review the log file, and execute the preupgrade fixup.


**upgrade**

once all pre-check are ok, we can do the actual update. The idea is to disable the broker configuration and to keep the standby shutdown (it is probably possible to keep the standby in apply-mode but it did not work for me). Then run the upgrade assistant on the primary and at the end restart the standby in the new home then enable the broker


on the standby - evs-rv-orarac03 -

disable the broker configuration

```
dgmgrl
connect sys/evs123
disable configuration;
```

disable the dg broker on both the primary and the dataguard

```
ALTER SYSTEM SET DG_BROKER_START=FALSE scope=both sid='*';
```

stop the database and stop the listener on the standby, then install the files in the new home

```
lsrnctl stop
sqlplus /nolog <<EOF
connect / as sysdba
shutdown immediate;
exit;
EOF
```

```
cd /u01/app/oracle/product/12.1.0
cp ./db_home1/dbs/spfileDEVRACDB3.ora ./db_home2/dbs/
cp ./db_home1/dbs/orapwDEVRACDB3 ./db_home2/dbs
cp ./db_home1/network/admin/*.ora ./db_home2/network/admin/
```

edit the listener.ora and change the oracle home
edit /etc/oratab and change the oracle home

on the primary - evs-rv-orarac01 -

stop all dbms_scheduler jobs, comment out cron entries (i.e. backup)

as application owner

```
sqlplus myuser/mypasswd
set pages 0 lines 240 trimspool on
spool dodisable.sql
select q'[exec dbms_scheduler.disable(name=>']'||
      job_name||
      q'[', force=>true);]'
from user_scheduler_jobs
where state='ENABLED';
```

review then execute dodisable.sql

```
sqlplus myuser/mypasswd
@dodisable.sql
```

purge the recycle bin

```
purge dba recycle;
```

turn off flashback

```
srvctl stop database -db DEVRACDB_MN
srvctl start instance -db DEVRACDB_MN -instance DEVRACDB1 -startoption nomount
```

```
sqlplus /nolog
connect / as sysdba
alter database flashback off;
exit;
```

```
srvctl stop instance -db DEVRACDB_MN -instance DEVRACDB1
srvctl start database -db DEVRACDB_MN
```

from the new home run dbua

after dbua, on the primary, you MUST copy back the tsnnames.ora from the old home to the new. After that re-enable the broker configuration (after starting the broker of course).

```
alter system set dg_broker_start=true scope=both sid='*';
enable configuration;
```

do not forget to turn flashback on again and to re-enable the jobs that were disabled before the upgrade.
