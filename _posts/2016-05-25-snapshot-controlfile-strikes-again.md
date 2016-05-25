---
layout: post
title: Snapshot Controlfile Strikes Again
comments: true
---

It's not the first time when the RMAN snapshot controlfile  feature causes
problems. I blogged about this in an [old
post](http://oracle-cookies.blogspot.ro/2009/11/strange-rman-snapshot-controlfile-issue.html).

It starts with a funny message when you want to drop the obsolete backups:

    RMAN-06207: WARNING: 1 objects could not be deleted for DISK channel(s) due
    RMAN-06208:          to mismatched status.  Use CROSSCHECK command to fix status
    RMAN-06210: List of Mismatched objects
    RMAN-06211: ==========================
    RMAN-06212:   Object Type   Filename/Handle
    RMAN-06213: --------------- ---------------------------------------------------
    RMAN-06214: Datafile Copy   /u01/app/oracle/product/11.2.0/db_11204/dbs/snapcf_volp1.f

In the case above, the message comes from a daily backup script which also takes
care of deleting the obsolete backups. However, the target database is a RAC one
and this snapshot controlfile needs to sit on a shared location. So, we
fixed the issue by simply changing the location to ASM:

    RMAN> CONFIGURE SNAPSHOT CONTROLFILE NAME TO '+DG_FRA/snapcf_volp.f';

    old RMAN configuration parameters:
    CONFIGURE SNAPSHOT CONTROLFILE NAME TO '/u01/app/oracle/product/11.2.0/db_11204/dbs/snapcf_volp1.f';
    new RMAN configuration parameters:
    CONFIGURE SNAPSHOT CONTROLFILE NAME TO '+DG_FRA/snapcf_volp.f';
    new RMAN configuration parameters are successfully stored

Ok, now let's get rid of that
`/u01/app/oracle/product/11.2.0/db_11204/dbs/snapcf_volp1.f` file. By the way,
have you noticed that it is considered to be a datafile copy instead of a
controlfile copy? Whatever...

The first try was:

    RMAN> delete expired copy;

    released channel: ORA_DISK_1
    allocated channel: ORA_DISK_1
    channel ORA_DISK_1: SID=252 instance=volp1 device type=DISK
    specification does not match any datafile copy in the repository
    specification does not match any archived log in the repository

    List of Control File Copies
    ===========================

    Key     S Completion Time     Ckp SCN    Ckp Time
    ------- - ------------------- ---------- -------------------
    42      X 2016-05-14 16:48:48 3152170591 2016-05-14 16:48:48
            Name: /u01/app/oracle/product/11.2.0/db_11204/dbs/snapcf_volp1.f
            Tag: TAG20160514T164847

    Do you really want to delete the above objects (enter YES or NO)? YES
    RMAN-00571: ===========================================================
    RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
    RMAN-00571: ===========================================================
    RMAN-03009: failure of delete command on ORA_DISK_1 channel at 05/25/2016 08:35:09
    ORA-19606: Cannot copy or restore to snapshot control file

Ups, WTF? I don't want to copy or restore anyting! Whatever... Next try:

    RMAN> crosscheck controlfilecopy '/u01/app/oracle/product/11.2.0/db_11204/dbs/snapcf_volp1.f';

    released channel: ORA_DISK_1
    allocated channel: ORA_DISK_1
    channel ORA_DISK_1: SID=252 instance=volp1 device type=DISK
    validation failed for control file copy
    control file copy file name=/u01/app/oracle/product/11.2.0/db_11204/dbs/snapcf_volp1.f RECID=42 STAMP=911839728
    Crosschecked 1 objects

    RMAN> delete expired controlfilecopy '/u01/app/oracle/product/11.2.0/db_11204/dbs/snapcf_volp1.f';


    released channel: ORA_DISK_1
    allocated channel: ORA_DISK_1
    channel ORA_DISK_1: SID=497 instance=volp1 device type=DISK

    List of Control File Copies
    ===========================

    Key     S Completion Time     Ckp SCN    Ckp Time
    ------- - ------------------- ---------- -------------------
    42      X 2016-05-14 16:48:48 3152170591 2016-05-14 16:48:48
            Name: /u01/app/oracle/product/11.2.0/db_11204/dbs/snapcf_volp1.f
            Tag: TAG20160514T164847


    Do you really want to delete the above objects (enter YES or NO)? yes
    RMAN-00571: ===========================================================
    RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
    RMAN-00571: ===========================================================
    RMAN-03009: failure of delete command on ORA_DISK_1 channel at 05/25/2016 08:41:11
    ORA-19606: Cannot copy or restore to snapshot control file

The same error! Even trying with the backup key wasn't helpful:

    RMAN> delete controlfilecopy 42;

    released channel: ORA_DISK_1
    allocated channel: ORA_DISK_1
    channel ORA_DISK_1: SID=252 instance=volp1 device type=DISK

    List of Control File Copies
    ===========================

    Key     S Completion Time     Ckp SCN    Ckp Time
    ------- - ------------------- ---------- -------------------
    42      X 2016-05-14 16:48:48 3152170591 2016-05-14 16:48:48
            Name: /u01/app/oracle/product/11.2.0/db_11204/dbs/snapcf_volp1.f
            Tag: TAG20160514T164847

    Do you really want to delete the above objects (enter YES or NO)? yes

    RMAN-00571: ===========================================================
    RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
    RMAN-00571: ===========================================================
    RMAN-03009: failure of delete command on ORA_DISK_1 channel at 05/25/2016 08:38:30
    ORA-19606: Cannot copy or restore to snapshot control file

The fix was to restore the old location of the snapshot controlfile and then to
delete the expired copy:

    RMAN> configure snapshot controlfile name to '/u01/app/oracle/product/11.2.0/db_11204/dbs/snapcf_volp1.f';

    old RMAN configuration parameters:
    CONFIGURE SNAPSHOT CONTROLFILE NAME TO '+dg_fra/snapcf_volp.f';
    old RMAN configuration parameters:
    CONFIGURE SNAPSHOT CONTROLFILE NAME TO '+DG_FRA/snapcf_volp.f';
    new RMAN configuration parameters:
    CONFIGURE SNAPSHOT CONTROLFILE NAME TO '/u01/app/oracle/product/11.2.0/db_11204/dbs/snapcf_volp1.f';
    new RMAN configuration parameters are successfully stored


    RMAN> delete expired copy;

    allocated channel: ORA_DISK_1
    channel ORA_DISK_1: SID=788 instance=volp1 device type=DISK
    specification does not match any datafile copy in the repository
    specification does not match any archived log in the repository

    List of Control File Copies
    ===========================

    Key     S Completion Time     Ckp SCN    Ckp Time
    ------- - ------------------- ---------- -------------------
    42      X 2016-05-14 16:48:48 3152170591 2016-05-14 16:48:48
            Name: /u01/app/oracle/product/11.2.0/db_11204/dbs/snapcf_volp1.f
            Tag: TAG20160514T164847

    Do you really want to delete the above objects (enter YES or NO)? yes

    deleted control file copy
    control file copy file name=/u01/app/oracle/product/11.2.0/db_11204/dbs/snapcf_volp1.f RECID=42 STAMP=911839728
    Deleted 1 EXPIRED objects

Don't ask me why it worked. Of course, after we fixed the expired copy the
snapshot controlfile location was set to a shared disk.
