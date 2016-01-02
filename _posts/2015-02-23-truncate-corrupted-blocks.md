---
layout: post
title: TRUNCATE a Table with Corrupted Blocks
---

The customer reported that a block corruption error is logged in one of his
databases. This happened for a table owned by the application we maintain. The
error was:

    ORA-01578: ORACLE data block corrupted (file # 8, block # 52195)
    ORA-01110: data file 8: ...
    ORA-26040: Data block was loaded using the NOLOGGING option

First we thought that the error is received on the production, but, later on,
they have provided more details of the problem and, apparently, this error was
occurring on a duplicate database they are creating every day as part of their
DR strategy.

The Root Cause (aka RC)
=======================

Well, not very difficult to grasp the big picture. The scenario we had in mind
was:

  1. the corrupted block was belonging to a table which, indeed, was being
     loaded using, yes, SqlLoader with the `DIRECT` option enabled.
  2. every night the production database was backed up.
  3. after the backup, the SqlLoader job took place
  4. then, the night backup and the archives generated afterwards were used 
     to create a duplicate database in DR.
  5. in DR, BANG!!! When the `NOLOGGING` table is queried Oracle complains with an
     `ORA-01578` error

Validate It
===========

Let's prepare the playground. First of all, we create a dummy user and its
corresponding tablespace:

    SQL> create tablespace guineea_pig datafile size 50M;

    Tablespace created.

    SQL> grant create session, create table to guineea_pig identified by ***;

    Grant succeeded.

    SQL> alter user guineea_pig default tablespace guineea_pig quota unlimited on guineea_pig;

    User altered.

Then, we create two tables: `T1` and `T2`. We'll use them later to do a `NOLOGGING`
operation as a substitute for the SqlLoader.

    SQL> connect guineea_pig/xxx
    Connected.
    SQL> create table t1 as select * from all_objects;

    Table created.

    SQL> create table t2 nologging as select * from t1 where 1=2;

    Table created.

Before anything else, let's make a backup of the database (step 1 from the
scenario):

    RMAN> backup database include current controlfile plus archivelog;

    Starting backup at 2015-02-23 17:24:41
    current log archived
    using channel ORA_DISK_1
    channel ORA_DISK_1: starting archived log backup set
    
    ...

    Finished backup at 2015-02-23 17:26:21

The above is the backup we'll use to create the duplicated database. Now, let's
perform a `NOLOGGING` operation (corresponds to the SqlLoader step):

    SQL> insert /*+ APPEND */ into t2 select * from t1;

    66167 rows created.

    SQL> commit;

    Commit complete.

Ok, let's backup just the archives now:

    RMAN> backup archivelog all not backed up 1 times;

    Starting backup at 2015-02-23 17:38:48
    current log archived
    allocated channel: ORA_DISK_1
    channel ORA_DISK_1: SID=44 device type=DISK
    skipping archived logs of thread 1 from sequence 592 to 594; already backed up
    channel ORA_DISK_1: starting archived log backup set
    channel ORA_DISK_1: specifying archived log(s) in backup set
    input archived log thread=1 sequence=595 RECID=4 STAMP=872444243
    input archived log thread=1 sequence=596 RECID=5 STAMP=872444328
    channel ORA_DISK_1: starting piece 1 at 2015-02-23 17:38:48
    channel ORA_DISK_1: finished piece 1 at 2015-02-23 17:38:49
    piece handle=/u01/app/oracle/fast_recovery_area/VENA/backupset/2015_02_23/o1_mf_annnn_TAG20150223T173848_bgpld8s2_.bkp tag=TAG20150223T173848 comment=NONE
    channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
    Finished backup at 2015-02-23 17:38:49

Now, ready for creating the duplicate from the backups. In our tests, `VENA` is
the production database and `VENADUP` is the duplicate. The general flow for a
duplicate (pretty boring for a DBA who did this thousands of times before):

  1. create the parameter file:
  
    a. create a dummy /tmp/venadup.ini file:

        *.audit_file_dest='/u01/app/oracle/admin/venadup/adump'
        *.audit_trail='db'
        *.compatible='11.2.0.4.0'
        *.db_block_size=8192
        *.db_create_file_dest='/u01/app/oracle/oradata'
        *.db_domain=''
        *.db_name='venadup'
        *.db_uniqu_name='venadup'
        *.db_recovery_file_dest='/u01/app/oracle/fast_recovery_area'
        *.db_recovery_file_dest_size=53687091200
        *.diagnostic_dest='/u01/app/oracle'
        *.dispatchers='(PROTOCOL=TCP) (SERVICE=venadupXDB)'
        *.memory_target=1660944384
        *.open_cursors=800
        *.processes=150
        *.remote_login_passwordfile='EXCLUSIVE'
        *.undo_tablespace='UNDOTBS1'

    b. create the SPFILE:

        [oracle@venus ~]$ ORACLE_SID=venadup sqlplus / as sysdba

        SQL*Plus: Release 11.2.0.4.0 Production on Mon Feb 23 17:56:12 2015

        Copyright (c) 1982, 2013, Oracle.  All rights reserved.

        Connected to an idle instance.

        SQL> startup pfile='/tmp/venadup.ini' nomount;
        ORACLE instance started.

        Total System Global Area 1653518336 bytes
        Fixed Size                  2253784 bytes
        Variable Size            1006636072 bytes
        Database Buffers          637534208 bytes
        Redo Buffers                7094272 bytes
        SQL> create spfile from pfile='/tmp/venadup.ini';

        File created.

    c. restart the instance with the new SPFILE:

        [oracle@venus ~]$ ORACLE_SID=venadup sqlplus / as sysdba

        SQL> shutdown immediate
        ORA-01507: database not mounted


        ORACLE instance shut down.
        SQL> startup nomount
        ORACLE instance started.

  2. create the password file for the duplicate database:

        [oracle@venus ~]$ cd $ORACLE_HOME/dbs
        [oracle@venus dbs]$ orapwd file=orapwvenadup password=****

  3. prepare the listener entry:

        SID_LIST_LISTENER =
          (SID_LIST =
            (SID_DESC =
              (GLOBAL_DBNAME = venadup)
              (ORACLE_HOME = /u01/app/oracle/product/11.2.0/4e_db)
              (SID_NAME = venadup)
            )

  4. reload the listener: ``lsnrctl reload``
  
  5. prepare the TNS entries:

        VENA =
          (DESCRIPTION =
            (ADDRESS_LIST =
              (ADDRESS = (PROTOCOL = TCP)(HOST = venus.localnet)(PORT = 1521))
            )
            (CONNECT_DATA =
              (SERVICE_NAME = VENA)
            )
          )

        VENADUP =
          (DESCRIPTION =
            (ADDRESS_LIST =
              (ADDRESS = (PROTOCOL = TCP)(HOST = venus.localnet)(PORT = 1521))
            )
            (CONNECT_DATA =
              (SERVICE_NAME = VENADUP)
            )
          )

  6. start the duplicate process:

        [oracle@venus dbs]$ rman target=sys/****@vena auxiliary=sys/****@venadup

        Recovery Manager: Release 11.2.0.4.0 - Production on Mon Feb 23 18:31:16 2015

        Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

        connected to target database: VENA (DBID=2950581980)
        connected to auxiliary database: VENADUP (not mounted)

        RMAN> duplicate target database to venadup;

Now, go and query the `T2` table:

    [oracle@venus dbs]$ sqlplus guineea_pig/****@venadup

    SQL> select count(*) from t2;
    select count(*) from t2
                        *
    ERROR at line 1:
    ORA-01578: ORACLE data block corrupted (file # 7, block # 1155)
    ORA-01110: data file 7:
    '/u01/app/oracle/oradata/VENADUP/datafile/o1_mf_guineea__bgpop6gz_.dbf'
    ORA-26040: Data block was loaded using the NOLOGGING option

Good, the initial hypothesis is confirmed.

Solutions?
==========

Well, obvious, the recommended way. Take a backup (full or incremental) of those
data files affected by the direct load operations and then do the duplicate. Or,
use the FORCE LOGGING on the database level, but this will affect the
performance of the load operation.

Back to the Question
====================

It turned out that the table in question was configured with `NOLOGGING` option
because it's just a stage table. So, after the records are processed, they are
not needed anymore in this table. That's why the title of this post: would be
possible to `TRUNCATE` the stage table on production after the processing is
finished so that, on the duplicate, querying this table to not pose any problems
as it is supposed to be empty? I forgot to mention, the biggest problem for the
customer was the flooding of the alert.log with that corruption error, triggered
by the job which gathers statistics automatically.

Actually we want to test this:

  1. we have the night backup
  2. then we have some `NOLOGGING` operations which doesn't generate redolog
  3. then a `TRUNCATE TABLE` is executed by the application at the end of the
     processing it's doing. This statement generates redolog for the command
     itself.
  4. later on, the duplicate takes place. It restores the night backup and
     applies the redolog generated after the backup was taken. The tricky thing
     is that, on the duplicate database, the table in question becomes corrupted
     because of the lack of the necessary redolog (those DIRECT LOAD
     operations), but a bit further in the redolog stream which is being
     crunched, the `TRUNCATE TABLE` command is applied on the corrupted table.
     Will it work or will it fail miserably?

Let's see. We assume that we have the night backup already taken and `T2` table
was affected by a direct load operation.

    [oracle@venus ~]$ sqlplus guineea_pig/*****@vena

    SQL> select count(*) from t2;

      COUNT(*)
    ----------
        66167

    SQL> truncate table t2;

    Table truncated.

    [oracle@venus ~]$ rman target /

    RMAN> backup archivelog all not backed up 1 times;

Now, duplicate the database and query the `T2` table:

    [oracle@venus ~]$ rman target=sys/****@vena auxiliary=sys/****@venadup

    Recovery Manager: Release 11.2.0.4.0 - Production on Mon Feb 23 18:55:43 2015

    Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

    connected to target database: VENA (DBID=2950581980)
    connected to auxiliary database: VENADUP (not mounted)

    RMAN> duplicate target database to venadup;

    [oracle@venus ~]$ sqlplus guineea_pig/*****@venadup

    SQL> select count(*) from t2;

      COUNT(*)
    ----------
            0

So, the answer is YES, a truncate will work on a table with corrupted
blocks because of a direct load operations.

Actually, it's working the other way around as well, on the duplicated database:

    [oracle@venus ~]$ sqlplus guineea_pig/***@venadup

    SQL*Plus: Release 11.2.0.4.0 Production on Mon Feb 23 19:51:32 2015

    Copyright (c) 1982, 2013, Oracle.  All rights reserved.


    Connected to:
    Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
    With the Partitioning option

    SQL> select count(*) from t2;
    select count(*) from t2
                        *
    ERROR at line 1:
    ORA-01578: ORACLE data block corrupted (file # 7, block # 1155)
    ORA-01110: data file 7:
    '/u01/app/oracle/oradata/VENADUP/datafile/o1_mf_guineea__bgpt3506_.dbf'
    ORA-26040: Data block was loaded using the NOLOGGING option


    SQL> truncate table t2;

    Table truncated.

    SQL> select count(*) from t2;

      COUNT(*)
    ----------
            0

So, in order to fix the problem, the DBA can safely truncate this table in the
duplicate database, as soon as the duplicate process is finished. Again, this is
just because we're talking about a staging table. Yet, a word of caution needs
to be raised: if the duplicate happens while the staging table is being
processed then the duplicate database might be inconsistent as far as the
application logic is concerned.

