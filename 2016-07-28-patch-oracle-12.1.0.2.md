## Patching oracle

After installing a RAC with its standby database, doing the upgrade from 12.1.0.1 to 12.1.0.2, now it is time to apply the latest patch set updates. I am not sure how much important patching is and I am wondering if it is worth the trouble. Indeed patching a RAC with a dataguard requires a lot of preparation upfront.

Firt of all head to this document on metaling: [Oracle Recommended Patches -- Oracle Database (Doc ID 756671.1)](Oracle Recommended Patches -- Oracle Database (Doc ID 756671.1))

Basically, in my case of a RAC 12.1.0.2, as of july 2016, I need the GSI patch set (patch for GRID Home), the database Patch set (patch for Oracle home) and the OJVM patch set (for Oracle home only, not needed on Grid home). I could download a bundle with the 3 of them, however there is a catch: the OJVM patch is a non-rolling patch and it requires a full shutdown of everyting on the cluser: therefore the way I am familiar with, using opatchauto, will not work as usual.

So bottom line: I will apply a bundle patch for GI PSU and DB PSU with opatchauto (it will do a rolling patch update) and after I will apply the OJVM patch with a complete downtime. It is obviously not the optimal way.



Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.
