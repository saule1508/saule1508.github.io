## Patching oracle

After installing a RAC with its standby database, doing the upgrade from 12.1.0.1 to 12.1.0.2, now it is time to apply the latest patch set updates. I am not sure how much important patching is and I am wondering if it is worth the trouble. Indeed patching a RAC with a dataguard requires a lot of preparation upfront.

Firt of all head to this document on metaling: [Oracle Recommended Patches -- Oracle Database (Doc ID 756671.1)](Oracle Recommended Patches -- Oracle Database (Doc ID 756671.1))

Basically, in my case of a RAC 12.1.0.2, as of july 2016, I need the GSI patch set (patch for GRID Home), the database Patch set (patch for Oracle home) and the OJVM patch set (for Oracle home only, not needed on Grid home). I could download a bundle with the 3 of them, however there is a catch: the OJVM patch is a non-rolling patch and it requires a full shutdown of everyting on the cluser: therefore the way I am familiar with, using opatchauto, will not work as usual.

So bottom line: I will apply a bundle patch for GI PSU and DB PSU with opatchauto (it will do a rolling patch update) and after I will apply the OJVM patch with a complete downtime. It is obviously not the optimal way.

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






Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.
