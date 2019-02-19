---
layout: post
title: em express and Oracle Restart 18
published: true
---

Yesterday I installed Oracle 18c with ASM, which implies to install the grid infrastructure software and then the usual Oracle software. This combination of grid + oracle on a standalone server (i.e. not a RAC) is called Oracle restart. 

Everything went fine but out of the box em express was not working. It turned out it was due to a permission issue on a wallet file and it was not easy to troubleshoot. It happened because I used different users for grid and for oracle, which is quite standard.

First thing is to check if em express is configured: connect to the Oracle instance (not to the +ASM instance !) and execute the function dbms_xdb_config.gethttpsport. It should not run zero
```
select dbms_xdb_config.GETHTTPSPORT() from dual;
```

in my case it is 5500 (the default). If it returns zero then use the function sethttpsport to set it up.

Then execute lsnrctl status to check that the listener is listening on this port

```bash
lsnrctl status
```
it showed the following line:
```
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcps)(HOST=XXX-r-ora02)(PORT=5500))(Security=(my_wallet_directory=/u01/app/oracle/product/18.0.0/dbhome_1/admin/MADROOT/xdb_wallet))(Presentation=HTTP)(Session=RAW))

```
and so accessing the URL XXX-r-ora02:5500/em in a browser should have worked (assuming the firewall port is open, if not open it)
```bash
sudo firewall-cmd --add-port 5500/tcp --permanent
sudo systemctl restart firewalld
```
But in my case it was not working and there was nothing in the listener.log. 

It turned out that in an Oracle restart set-up the listener is started by operating system grid but the wallet file was not readable by user grid:

```
ls -al /u01/app/oracle/product/18.0.0/dbhome_1/admin/MADROOT/xdb_wallet/*
-rw-------. 1 oracle asmadmin 3880 Feb 18 17:05 /u01/app/oracle/product/18.0.0/dbhome_1/admin/MADROOT/xdb_wallet/cwallet.sso
-rw-------. 1 oracle asmadmin 3835 Feb 18 17:05 /u01/app/oracle/product/18.0.0/dbhome_1/admin/MADROOT/xdb_wallet/ewallet.p12
```
and the solution was to give read access to the group asmadmin
```bash
chmod g+r /u01/app/oracle/product/18.0.0/dbhome_1/admin/MADROOT/xdb_wallet/*
```

Note that there is still a certificate warning when using the https url, this is because it uses a self-signed certificate. You can enable the http port if you want:
```sql
SQL> select dbms_xdb_config.GETHTTPport() from dual;

DBMS_XDB_CONFIG.GETHTTPPORT()
-----------------------------
                            0

SQL> exec dbms_xdb_config.sethttpport(8080);

PL/SQL procedure successfully completed.

SQL> select dbms_xdb_config.GETHTTPport() from dual;

DBMS_XDB_CONFIG.GETHTTPPORT()
-----------------------------
                         8080

```

