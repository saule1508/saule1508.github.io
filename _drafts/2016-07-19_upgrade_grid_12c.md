# Upgrade grid from 12.1.0.1 to 12.1.0.2

I have a RAC set-up with two nodes and grid 12.1.0.1 on it. I need to upgrade it to 12.1.0.2.

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


