---
layout: post
title: Patching Oracle RAC 12c with latest PSU
published: true
---

After installing a RAC with its standby database, doing the upgrade from 12.1.0.1 to 12.1.0.2, now it is time to apply the latest patch set updates. Patching a RAC with a dataguard requires a lot of preparation upfront, so I am not planning to do this every 3 months.

Firt of all head to this document on metaling: [Oracle Recommended Patches -- Oracle Database (Doc ID 756671.1)](Oracle Recommended Patches -- Oracle Database (Doc ID 756671.1))

Basically, in my case of a RAC 12.1.0.2, as of july 2016, I need the GSI patch set (patch for GRID Home), the database Patch set (patch for Oracle home) and the OJVM patch set (for Oracle home only, not needed on Grid home). I could download a bundle with the 3 of them, however there is a catch: the OJVM patch is a non-rolling patch and it requires a full shutdown of everyting on the cluser: therefore the way I am familiar with, using opatchauto, will not work as usual.

So bottom line: I will apply a bundle patch for GI PSU and DB PSU with opatchauto (it will do a rolling patch update) and after I will apply the OJVM patch with a complete downtime. It is obviously not the optimal way.

so I have two of them:
-12.1.0.2.160719 (Jul 2016) Grid Infrastructure Patch Set Update (GI PSU): 23273629
-OJVM PATCH SET UPDATE 12.1.0.2.160719: 23177536

but for the dataguard, since it is a single instance database (not RAC), I need 
- 12.1.0.2.160719 (Jul 2016) Database Patch Set Update (DB PSU): 23054246
-OJVM PATCH SET UPDATE 12.1.0.2.160719: 23177536

The grid infrastructure PSU, 23273629, will patch both the GI HOME and the DB HOME. It must be executed on one node after the other.

The OJVM PSU is a bit involved on the RAC, it requires a complete downtime and it requires to run datapatch in startup upgrade mode with the parameter cluster_database set to false (this is all explained in the readme).

**note about dataguard**: for the standby my strategy is the following. First set transport-off and apply-off using the broker (dgmgrl command line). Then stop the db and the listener and apply both patches but don't run the datapatch utility on the standby. Keep the db stopped, it will be restarted - in mount mode - when the primary is fully patched. At this time startup mount the DB (mount, not yet open), set transport-on and apply-on so that the standby will recover from the primary. When this is done the standby can be opened (its state will become read only with apply)


===opatch utility

first think: get the latest version of opatch for your version. In this case the file is p6880880_121010_Linux-x86-64.zip and, strangely, it contains opatch version 12.2.0.1.5. It looks like there are some improvements in this version of OPatch, especially the response file is not needed anymore. But the  documentation is not completly up to date and all the blogs still mention this response file which is a bit confusing

```
scp opatch/p6880880_121010_Linux-x86-64.zip oracle@evs-rv-orarac01:/u01/staging/opatch/
cd $ORACLE_HOME
mv OPatch OPatch.old
unzip /u01/staging/opatch/p6880880_121010_Linux-x86-64.zip
./OPatch/opatch version
```

this must be done on all nodes in the ORACLE_HOME (with user oracle) and in the GRID_HOME (this time with user grid)

for user grid it is a bit more complex because the user has no write right on the GRID_HOME, so I cannot unzip the file as user grid. So I did it with user root, then set ownership and permission similar to the current OPatch

as user root

```
cd /u01/app/12.1.0.2/grid
mv OPatch OPatch.old
unzip /u01/staging/opatch/p6880880_121010_Linux-x86-64.zip
chown grid:oinstall -R OPatch
chmod 766 OPatch
```

as user grid

```
$ORACLE_HOME/OPatch/opatch version
```
it says 12.2.0.1.5

now let's move on to PSU 

===12.1.0.2.160719 (Jul 2016) Grid Infrastructure Patch Set Update (GI PSU): 23273629

as grid and as oracle, run the following

```
$ORACLE_HOME/OPatch/opatch lsinventory -detail -oh $ORACLE_HOME
```
NB: it will create a text file in $ORACLE_HOME/cfgtoollogs/opatch/lsinv

as user grid, unzip the file p23273629_121020_Linux-x86-64.zip in /u01/staging/patch_grid.

```
unzip p23273629_121020_Linux-x86-64.zip 
```

as user root run opatch with -analyze flag: this will only test the patch

```
. /usr/local/bin/oraenv
export PATH=$PATH:$ORACLE_HOME/OPatch
opatchauto apply /u01/staging/patch_grid/23273629 -analyze
```
Do that on the other node also.

Luckily on metalink I saw there is a bug with the database part of the patch, the work-around is well documented. This shows how important it is to read all documentation and readme upfront.

The note documenting the bug is: Doc ID 2163593.1. The work-around implies installing unixODBC and unixODBC-devel then do a relink inside oracle home. There is an alternative work-around but it did not work for me.

as user root, on node 1

```
# opatchauto apply /u01/staging/patch_grid/23273629
```
this takes time and at it end it says that four patches were successfully applied


as user root, on node 2

```
# opatchauto apply /u01/staging/patch_grid/23273629
```
this time you'll see a message saying that it is applying the SQL patches on the RAC home, i.e. because it is the last node it will run datapatch

check the install with opatch lsinventory and with the following sql

```
select patch_uid,version,action,status,ACTION_TIME,description from dba_registry_sqlpatch;
```

===OJVM PATCH SET UPDATE 12.1.0.2.160719: 23177536

this patch is not rolling and requires a full downtime

get p23177536_121020_Linux-x86-64.zip into /u01/staging/patch_jvm on all nodes (user oracle) and unzip, then run

```
$ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail -ph /u01/staging/patch_jvm/23177536
```
stop database and all services running on ORACLE_HOME

```
srvctl stop database -db DEVRACDB_MN
```

as root

```
crsctl stop cluster -all
```

then check the status of crsd with and wait until it is down

```
crsctl status resource -t -init
```

on both nodes I also did (probably not needed)

```
crsctl stop crs
```

whith ps I see there is nothing running in the ORACLE_HOME (there is still one process in the GRID_HOME though, a java process running the class oracle.rat.tfa.TFAMain. It is ok because the GRID_HOME will not be patched.

On node 1, as user oracle, I can apply the patch

```
cd /u01/staging/patch_jvm/23177536
$ORACLE_HOME/OPatch/opatch apply
```

it will apply the patch on both nodes.

Now we must run datapatch, which is a bit tricky on a RAC

As user oracle, on node 1

```
sqlplus /nolog
connect / as sysdba
startup 
alter system set cluster_database=false scope=spfile;
exit;
srvctl stop database -db DEVRACDB_MN
sqlplus /nolog
connect / as sysdba
startup upgrade
```

now we can run datapatch
```
cd $ORACLE_HOME/OPatch
./datapatch -verbose
```

```
sqlplus /nolog
connect / as sysdba
alter system set cluster_database=true scope=spfile;
shutdown;
```

and finally we can restart all

first restart crs on the second node (it was stopped, probably for no good reason)

```
crsctl start crs
```

then restart the database

```
svrctl start database -db DEVRACDB_MN
```
then check in the dba_registry_sqlpatch view;

```
select PATCH_ID, PATCH_UID, VERSION, STATUS, DESCRIPTION
from DBA_REGISTRY_SQLPATCH
order by BUNDLE_SERIES;
```

don't forget to take care of the dataguard, as mentionned above.


