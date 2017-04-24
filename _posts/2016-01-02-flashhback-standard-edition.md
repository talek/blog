---
layout: post
title: Flashback Tablespace
comments: true
---
We want to easily flashback just a part of the database. We need this for our
developers which are sharing the same database, but using just their allocated
schema and tablespaces. They want to be allowed to set a restore point (on the
schema level), to run their workload and then to be able to revert back their
schema as it was when the restore point was taken. The other users on the
database must not be disrupted in any way.

Of course, flashback database feature is not good enough because it reverts back
the entire database and can't be used without downtime. On the other hand, a
TSPITR (Tablespace point in time recovery) on big data-files is a PITA from the
speed point of view, especially because we need to copy those big files when the
transportable set is created and, again, as part of the restore, when we need to
revert back.

<s>In addition, we want this to work even on an Oracle standard edition.</s>

## The Idea

Use a combination of TSPITR and a file system with snapshotting capabilities. As
a proof of concept, we'll use [btrfs](https://en.wikipedia.org/wiki/Btrfs), but
it can be [zfs](https://en.wikipedia.org/wiki/ZFS) or other exotic file systems.

Below is how the physical layout of the database should be designed.

![]({{ site.baseurl }}assets/images/flashback_oracle_se.png "Design")

As you can see, the non-application datafiles, or those which do not need any
flashback functionality, may reside on a regular file system. However, the
datafiles belonging to the applications which need the "time machine", have to
be created in a sub-volume of the `btrfs` pool. For each sub-volume we can
create various snapshots, as shown in the above figure: `app1-s1`, `app1-s2`
etc.

To create a restore point for the application tablespaces we'll do the following:

  1. put the tablespaces for the application in read only mode (FAST, we assume
     that no long-running/pending transactions are active)
  2. create a transportable set for those tablespace. So, we end up with a
     dumpfile having the metadata for the above read only tablespaces. (pretty
     FAST, only segment definitions are exported)
  3. export all non-tablespace database objects (sequences, synonyms etc.) for
     the target schema. (pretty FAST, no data, just definitions)
  4. take a snapshot of the sub-volume assigned to the application. That
     snapshot contain the read-only datafiles, the dump file from the
     transportable set and the dump file with all the non-tablespace
     definitions. (FAST, because we rely on the snapshotting feature which is
     fast by design)
  5. put the tablespaces back in read only mode (FAST).

For a flashback/revert, the steps are:

  1. delete the application user and its tablespaces (pretty FAST. it depends on
     how many objects needs to be dropped)
  2. switch to snapshot (FAST, by design)
  3. reimport user from the non-tablespace dumpfile (FAST)
  4. re-attached the read only tablespaces from the transportable set (FAST)
  5. reimport the non-tablespace objects (FAST, just definitions)
  5. put the tablespaces back in read write (FAST)

## Playground Setup

On OL6.X we need to install this:

    yum install btrfs-progs

Next, create the main volume:

    [root@venus ~]# fdisk -l

    Disk /dev/sda: 52.4 GB, 52428800000 bytes
    255 heads, 63 sectors/track, 6374 cylinders
    Units = cylinders of 16065 * 512 = 8225280 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk identifier: 0x0003cb01

       Device Boot      Start         End      Blocks   Id  System
    /dev/sda1   *           1        6375    51198976   83  Linux

    Disk /dev/sdb: 12.9 GB, 12884901888 bytes
    255 heads, 63 sectors/track, 1566 cylinders
    Units = cylinders of 16065 * 512 = 8225280 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk identifier: 0x00000000


    Disk /dev/sdc: 12.9 GB, 12884901888 bytes
    255 heads, 63 sectors/track, 1566 cylinders
    Units = cylinders of 16065 * 512 = 8225280 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk identifier: 0x00000000

The disks we're going to add are: `sdb` and `sdc`.

    [root@venus ~]# mkfs.btrfs /dev/sdb /dev/sdc

    WARNING! - Btrfs v0.20-rc1 IS EXPERIMENTAL
    WARNING! - see http://btrfs.wiki.kernel.org before using

    failed to open /dev/btrfs-control skipping device registration: No such file or directory
    adding device /dev/sdc id 2
    failed to open /dev/btrfs-control skipping device registration: No such file or directory
    fs created label (null) on /dev/sdb
            nodesize 4096 leafsize 4096 sectorsize 4096 size 24.00GB
    Btrfs v0.20-rc1

Prepare the folder into which to mount to:

    [root@venus ~]# mkdir /oradb
    [root@venus ~]# chown -R oracle:dba /oradb

Mount the btrfs volume:

    [root@venus ~]# btrfs filesystem show /dev/sdb
    Label: none  uuid: d30f9345-74bc-42df-a55d-f22e3a9c6e78
            Total devices 2 FS bytes used 28.00KB
            devid    2 size 12.00GB used 2.01GB path /dev/sdc
            devid    1 size 12.00GB used 2.03GB path /dev/sdb

    [root@venus ~]# mount -t btrfs /dev/sdc /oradb
    [root@venus ~]# df -h
    Filesystem            Size  Used Avail Use% Mounted on
    /dev/sda1              49G   39G  7.6G  84% /
    tmpfs                 2.0G     0  2.0G   0% /dev/shm
    bazar                 688G  426G  263G  62% /media/sf_bazar
    /dev/sdc               24G   56K   22G   1% /oradb

Create a new sub-volume to host the datafile for a developer schema.

    [root@venus /]# btrfs subvolume create /oradb/app1
    Create subvolume '/oradb/app1'
    [root@venus /]# chown -R oracle:dba /oradb/app1

In Oracle, create a new user with a tablespace in the above volume:

    create tablespace app1_tbs
      datafile '/oradb/app1/app1_datafile_01.dbf' size 100M;

    create user app1 identified by app1
    default tablespace app1_tbs
    quota unlimited on app1_tbs;

    grant create session, create table, create sequence to app1;

We'll also need to create a directory to point to the application folder. It is
need for `expdp` and `impdp`.

    CREATE OR REPLACE DIRECTORY app1_dir AS '/oradb/app1';

Let's create some database objects:

    connect app1/app1
    create sequence test1_seq;
    select test1_seq.nextval from dual;
    select test1_seq.nextval from dual;
    create table test1 (c1 integer);
    insert into test1 select level from dual connect by level <= 10;
    commit;

## Create a Restore Point

Now, we want to make a restore point. We'll need to:

  1. put the tablespace in read-only mode:

          alter tablespace app1_tbs read only;

  2. export tablespace:

          [oracle@venus app1]$ expdp userid=talek/*** directory=app1_dir transport_tablespaces=app1_tbs dumpfile=app1_tbs_metadata.dmp logfile=app1_tbs_metadata.log

  3. export the other database objects which belong to the schema but they are
     not stored in the tablespace:

          [oracle@venus app1]$ expdp userid=talek/*** directory=app1_dir schemas=app1 exclude=table dumpfile=app1_nontbs.dmp logfile=app_nontbs.log

  4. take a snapshot of the volume (this is very fast):

          [root@venus /]# btrfs subvolume list /oradb
          ID 258 gen 14 top level 5 path app1
          [root@venus app1]# btrfs subvolume snapshot /oradb/app1 /oradb/app1_restorepoint1
          Create a snapshot of '/oradb/app1' in '/oradb/app1_restorepoint1'

  5. we can get rid now of the dumps/logs:

          [root@venus app1]# rm -rf /oradb/app1/*.dmp
          [root@venus app1]# rm -rf /oradb/app1/*.log
          [root@venus app1]# ls -al /oradb/app1
          total 102412
          drwx------ 1 oracle dba             40 Nov 13 20:21 .
          drwxr-xr-x 1 root   root            44 Nov 13 20:02 ..
          -rw-r----- 1 oracle oinstall 104865792 Nov 13 20:13 app1_datafile_01.dbf

  6. Make the tablespace READ-WRITE again:

          alter tablespace app1_tbs read write;

## Simulate Workload

Now, let's simulate running a deployment script or something. We'll truncate the
TEST1 table and we'll create the TEST2 table. In addition, the sequence will be
dropped.

    20:25:04 SQL> select count(*) from test1;

    COUNT(*)
    --------
          10

    20:25:04 SQL> truncate table test1;

    Table truncated.

    20:25:29 SQL> select count(*) from test1;

    COUNT(*)
    --------
           0

    20:25:04 SQL> create table test2 (c1 integer, c2 varchar2(100));

    Table created.

    SQL> drop sequence test1_seq;

    Sequence dropped.

As you can see, all the above are DDL statements which cannot be easily
reverted.

## Revert to the Restore Point

Ok, so we need to revert back, and fast. The steps are:

  1. drop the tablespace:

          drop tablespace app1_tbs including contents;

  2. drop the user;

          drop user app1 cascade;

  2. revert to snapshot on the FS level:

          [root@venus oradb]# btrfs subvolume delete app1
          Delete subvolume '/oradb/app1'
          [root@venus oradb]# btrfs subvolume list /oradb
          ID 259 gen 15 top level 5 path app1_snapshots
          [root@venus oradb]# btrfs subvolume snapshot /oradb/app1_restorepoint1/ /oradb/app1
          Create a snapshot of '/oradb/app1_restorepoint1/' in '/oradb/app1'

  3. recreate the user with its non-tablespace objects. Please note that we
     remap the tablespace `APP1_TBS` to `USERS` because `APP1_TBS` is not
     created yet. Likewise, we exclude the quotas for the same reason.

          [oracle@venus scripts]$ impdp userid=talek/*** directory=app1_dir schemas=app1 remap_tablespace=app1tbs:users dumpfile=app1_nontbs.dmp nologfile=y exclude=tablespace_quota

  3. reimport the transportable set:

          [oracle@venus app1]$ impdp userid=talek/*** directory=app1_dir dumpfile=app1_tbs_metadata.dmp logfile=app1_tbs_metadata_imp.log transport_datafiles='/oradb/app1/app1_datafile_01.dbf'

  4. reimport quotas:

          [oracle@venus scripts]$ impdp userid=talek/*** directory=app1_dir schemas=app1 remap_tablespace=app1tbs:users dumpfile=app1_nontbs.dmp nologfile=y include=tablespace_quota

  5. restore the old value for the default tablespace of the APP1 user:

          alter user app1 default tablespace APP1_TBS;

  6. make the application tablespace read write:

          alter tablespace app1_tbs read write;

  7. remove the `/oradb/app1/*.dmp` and `/oradb/app1/*.log` files. They are not
     needed anymore.

## Test the Flashback

Now, if we connect with app1 we get:

    22:53:14 SQL> select object_name, object_type from user_objects;

    OBJECT_NAME OBJECT_TYPE
    ----------- -----------
    TEST1_SEQ   SEQUENCE
    TEST1       TABLE

    22:53:48 SQL> select count(*) from test1;

    COUNT(*)
    --------
          10

We can see that we are back on the previous state, across DDLs. The records from
the TEST1 table and our sequence are back.

## Remove an Un-necessary Snapshot

If the snapshot is not needed anymore it can be deleted with:

    [root@venus oradb]# btrfs subvolume delete /oradb/app1_restorepoint1
    Delete subvolume '/oradb/app1_restorepoint1'

## Pros & Cons

The main advantages for this solution are:

  * <s>no need for an Oracle enterprise edition. It's working with a standard
    edition without any problems.</s>
  * no need to have the database in `ARCHIVELOG`.
  * no flashback logs needed.
  * schema level flashback.
  * the database is kept online without any disruption for the other schema
    users.
  * can be easily scripted/automated
  * fast

The disadvantages:

  * `btrfs` is not quite production ready, but it is suitable for test/dev
    environments. For production systems `zfs` might be a better alternative.

