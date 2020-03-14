---
layout: post
title: Oracle cluster health framework 19
published: false
---

A lot of the support tools (orachk, OSWatcher,..) are now bundled together under the umbrella of the cluster health framework
<!--more-->

# OSWatcher

OSWatcher is now deployed as part of tfa (trace file analyzer).

```
[root@ora01 oswbb]# tfactl toolstatus

.------------------------------------------------------------------.
|                    TOOLS STATUS - HOST : ora01                   |
+----------------------+--------------+--------------+-------------+
| Tool Type            | Tool         | Version      | Status      |
+----------------------+--------------+--------------+-------------+
| Development Tools    | orachk       |   19.3.0.0.0 | DEPLOYED    |
|                      | oratop       |       14.1.2 | DEPLOYED    |
+----------------------+--------------+--------------+-------------+
| Support Tools Bundle | darda        | 2.10.0.R6036 | DEPLOYED    |
|                      | oswbb        |        8.3.2 | NOT RUNNING |
|                      | prw          | 12.1.13.11.4 | NOT RUNNING |
+----------------------+--------------+--------------+-------------+
| TFA Utilities        | alertsummary |   19.3.0.0.0 | DEPLOYED    |
|                      | calog        |   19.3.0.0.0 | DEPLOYED    |
|                      | dbcheck      |   18.3.0.0.0 | DEPLOYED    |
|                      | dbglevel     |   19.3.0.0.0 | DEPLOYED    |
|                      | grep         |   19.3.0.0.0 | DEPLOYED    |
|                      | history      |   19.3.0.0.0 | DEPLOYED    |
|                      | ls           |   19.3.0.0.0 | DEPLOYED    |
|                      | managelogs   |   19.3.0.0.0 | DEPLOYED    |
|                      | menu         |   19.3.0.0.0 | DEPLOYED    |
|                      | param        |   19.3.0.0.0 | DEPLOYED    |
|                      | ps           |   19.3.0.0.0 | DEPLOYED    |
|                      | pstack       |   19.3.0.0.0 | DEPLOYED    |
|                      | summary      |   19.3.0.0.0 | DEPLOYED    |
|                      | tail         |   19.3.0.0.0 | DEPLOYED    |
|                      | triage       |   19.3.0.0.0 | DEPLOYED    |
|                      | vi           |   19.3.0.0.0 | DEPLOYED    |
'----------------------+--------------+--------------+-------------'

Note :-
  DEPLOYED    : Installed and Available - To be configured or run interactively.
  NOT RUNNING : Configured and Available - Currently turned off interactively.
  RUNNING     : Configured and Available.
```

Here it is deployed but not running.

```
tfactl> start oswbb                                                                                                                                                                     
Starting OSWatcher
ERROR: Failed to start OSWatcher. Please review /oraapp01/oracle.ahf/data/repository/suptools/ora01/oswbb/oragrid/run_1578210850.log for details

tfactl> quit                                                                                                                                                                            
```

Even though it says failed to start OSWatcher, it is running and tfactl toolstatus shows it running

# Orachk

Again, it seems to be part of TFA since toolstatus says orachk deployed.

