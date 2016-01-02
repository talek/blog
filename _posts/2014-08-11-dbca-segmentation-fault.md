---
layout: post
title: DBCA segmentation fault
comments: true
---

Finally, I have succeeded to install the Oracle 12.1.0.2 database! Big deal,
you'd say! Well, yes! Usually, installing a new Oracle database on a fresh
server shouldn't be a problem: `./runInstaller`, next, next, software only, next
next, dbca... In my case there was no problem installing the software, but when
I was about to run the "dbca" tool, it always complained with something like
this:

    [oracle@mercur ~]$ dbca
    #
    # A fatal error has been detected by the Java Runtime Environment:
    #
    #  SIGSEGV (0xb) at pc=0x00007f7a21db777f, pid=2856, tid=140162988832512
    #
    # JRE version: 6.0_75-b13
    # Java VM: Java HotSpot(TM) 64-Bit Server VM (20.75-b01 mixed mode linux-amd64 compressed oops)
    # Problematic frame:
    # C  [libclntshcore.so.12.1+0x1ba77f]  long+0x8f
    #
    # An error report file with more information is saved as:
    # /home/oracle/hs_err_pid2856.log
    #
    # If you would like to submit a bug report, please visit:
    #   http://java.sun.com/webapps/bugreport/crash.jsp
    # The crash happened outside the Java Virtual Machine in native code.
    # See problematic frame for where to report the bug.
    #
    Aborted (core dumped)

Ouch! It turned out that the problem was my ``NLS_DATE_FORMAT`` environment
variable:

    [oracle@mercur ~]$ echo $NLS_DATE_FORMAT
    yyyy-mm-dd hh 24:mi:ss

Note that stupid blank between "hh" and "24". After I fixed the format my "dbca"
was happy again.
