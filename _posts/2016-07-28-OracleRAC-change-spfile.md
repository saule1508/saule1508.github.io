---
layout: post
title: Change Oracle spfile on a RAC
published: true
---

If you need to change system parameters in a RAC environment, you can of course use the usual alter system command, like for example:
<!--more-->

```
alter system set job_queue_processes=1000 scope=both sid='*';
```

but sometimes, if you need to change a lot of parameters or hidden parameters, it is easier to change them  all in the old init.ora text file.

First you need to find the location of the spfile. Use srvctl for that (as user oracle)

```
srvctl config
```

this gives you the name of the database, DEVRACDB_MN in my case

```
srvctl config -database DEVRACDB_MN
```

tells me that the spfile is +DATA/DEVRACDB_MN/spfiledevracdb_mn

now I can generate a text file from the spfile

```
sqlplus /nolog
connect / as sysdba
create pfile='/home/oracle/myinit.ora' from spfile='+DATA/DEVRACDB_MN/spfiledevracdb_mn';
exit;
```
edit the file /home/oracle/myinit.ora. Once done, we will stop the database, start one instance in mount mode and recreate the spfile from the text file

```
srvctl stop database -db DEVRACDB_MN
sqlplus /nolog
connect / as sysdba
startup nomount pfile='/home/oracle/myinit.ora';
create spfile='+DATA/DEVRACDB_MN/spfiledevracdb_mn' from pfile='/home/oracle/myinit.ora'
shutdown ;
exit;
```

and now we can restart the database with srvctl

```
srvctl start database -db DEVRACDB_MN
```

then use sqlplus show parameter command to see that the parameters are changed.


