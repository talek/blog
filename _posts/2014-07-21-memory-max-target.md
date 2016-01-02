---
layout: post
title: MEMORY_TARGET Not Supported on This System
comments: true
---

I wasn't aware, until this very moment, that there's a strong correlation
between the `MAX_MEMORY_TARGET` parameter and the `ORA-00845: MEMORY_TARGET not
supported on this system` error. I thought that the `MEMORY_TARGET` is all that
counts, considering that the actual memory allocation is based on this setting.
However, apparently that's not the case: Oracle will check the `MAX_MEMORY_TARGET`
too and will complain if the value of this parameter cannot be honored by the
available memory from the system.

The Proof
---------

Let's have a look on the following system:

    [oracle@ion ~]$ df -h
    Filesystem            Size  Used Avail Use% Mounted on
    /dev/sda1              49G   20G   27G  43% /
    tmpfs                 2.0G     0  2.0G   0% /dev/shm

As we can see, there are 2G of free shared memory in our tmpfs. Let's configure
our database with a `MEMORY_TARGET` of 1G and a `MAX_MEMORY_TARGET` of 4G.

    SQL> alter system set memory_target=1G scope=spfile;

    System altered.

    SQL> alter system set memory_max_target=4G scope=spfile;

    System altered.

Let's restart the instance:

    SQL> startup force
    ORA-00845: MEMORY_TARGET not supported on this system

Ups, it doesn't start up because the `MEMORY_MAX_TARGET` is set too high.

More Than One Database
----------------------

If there are many databases running on the same host then things start to get
interesting (kind of). I already have a database running:

    [oracle@ion ~]$ ps aux | grep pmon
    oracle    7161  0.0  0.4 1284760 19700 ?       Ss   12:21   0:00 ora_pmon_iondb

And the allocation in tmpfs is:

    [oracle@ion ~]$ df -h
    Filesystem            Size  Used Avail Use% Mounted on
    /dev/sda1              49G   20G   27G  43% /
    tmpfs                 2.0G  250M  1.7G  13% /dev/shm

Good! Let's start the second database on the same host, using a
`MEMORY_MAX_TARGET` of 1800M and a `MEMORY_TARGET` of 1500MB. I hand crafted these
parameters into a customized init.ora file.

    [oracle@ion ~]$ ORACLE_SID=iondb2 sqlplus / as sysdba

    SQL*Plus: Release 11.2.0.4.0 Production on Mon Jul 21 12:55:45 2014

    Copyright (c) 1982, 2013, Oracle.  All rights reserved.

    Connected to an idle instance.

    SQL> startup pfile='/home/oracle/init.ora'
    ORA-00845: MEMORY_TARGET not supported on this system

Pretty obvious, but please note that the value of the `MEMORY_MAX_TARGET` is
checked against the free available space from the tempfs, not against the total
size of the tempfs. This is important to keep in mind, especially because Oracle
doesn't allocate the whole memory from the very beginning, which means that you
might be able to startup both databases but, as they begin to allocate stuff in
their own SGA, you might end up with this `ORA-00845` error later.
