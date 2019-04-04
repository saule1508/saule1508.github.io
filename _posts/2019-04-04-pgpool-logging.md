---
layout: post
title: pgpool logging on centos 7
published: true
---

This is a quick post to explain logging of pgpool to the journal or to a file
<!--more-->

# pgpool logging configuration

In the pgool.conf file (/etc/pgpool-II/pgpool.conf), set the parameter log_destination to 'stderr' and debug_level to 0 (or more if you need)

```
log_destination='stderr'
debug_level = 0
```

With this the logs will be captured by journald and that's usually fine, journald will manage all the rest for you (no need to logrotate and you have the full power of journalctl to consul the logs).

```
journalctl --unit pgpool -f 
```
is the same as tail -f /var/log/pgpool.log

if you want to correlate the logs with those of postgres, it is easy, for example

```
journalctl --unit pgpool --unit postgresql --since -30min
```

This said sometimes you need the logs in a flat file, it is also possible. We need to modify the systemd pgpool unit

```
systemctl edit pgpool
```
this will open a file /etc/systemd/system/pgpool.service.d/override.conf

then put this into it

```
[Service]
User=postgres
Group=postgres
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=pgpool
```
NB: the User and Group settings are not related to logging, it is to force pgpool to be run as user postgres

When you modify a systemd unit file you must reload the config
```
systemd daemon-reload
```

then configure rsyslog by adding a file /etc/rsyslog.d/pgpool.conf

```
if ( $programname == "pgpool" ) then {
    action(type="omfile" fileOwner="postgres" fileGroup="postgres" fileCreateMode="0644" file="/var/log/pgpool.log")
    stop
}
````

and restart rsyslog and pgpool

```
systemctl restart rsyslog 
systemctl restart pgpool
```

Don't forget to add a logrotate configuration, usually adding a file /etc/logrotate.d/pgpool.conf, for example (I did not test this one)

```
/var/log/pgpool.log {
  create 0664 postgres postgres
  rotate 7
  missingok
  copytruncate
}
```