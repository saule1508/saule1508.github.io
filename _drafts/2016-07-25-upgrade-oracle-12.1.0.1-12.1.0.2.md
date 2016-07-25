---
published: false
---
## A New Post

I must upgrade a RAC production with a standby database (the standby is single-instance and non-ASM) from 12.1.0.1 to 12.1.0.2. It is a bit stupid because Oracle 12.2 is about to be released but I don't have the choice...

In order to do a rehearsal and learn the process I have set-up a RAC on VMWare ESX with a dataguard also on ESX. The grid infrastructure upgrade went smoothly (see previous blog post), now I'll document the database part.

The process is the following

- install the software on the standby (install soft only) and on the node 1 of the RAC (install soft only)
- run cluvfy
- run preupgrade scripts and do the fixup
- stop the apply on the standby, stop the standby, prepare the new home (copy spfile, password file, modifyl listener.ora, oratab etc.), then startup mount the standby with the new home. Restart the apply
- start dbua on the primary RAC instance 1

let's document those steps;

copy the files 

Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.
