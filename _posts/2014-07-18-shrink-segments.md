---
layout: post
title: Shrink Segments
---

I don't know why I haven't paid more attention to this feature. It is around
from Oracle 10g, yet I still defrag tables using the old school approach: ALTER
TABLE ... MOVE. Well, that's something I'm going to change! We need to shrink
some big tables on a production system and we come up with a rough estimation of
four hours of downtime, just to do this segment reorganization. Of course, the
users are not happy with this large downtime window (and when I say users I mean
the managers), therefore we need to do it with the least level of disruptions.
By the way, it's an Oracle standard edition. Forget online reorganization!

Things to be tested:

1. the space gain after shrinking
2. is it really an online operation? how does it work when there are concurrent
   sessions updating the target table?
3. are the triggers a problem if any are defined on the target table?
4. does it maintain the associated indexes?
5. does also shrink the LOB segments behind the columns?

Setup playground
----------------

Target DB is: ``Oracle Database 11g Release 11.1.0.6.0 - 64bit Production``

Create an Oracle user:

    create user talek identified by <pwd>;
    grant resource, connect to talek;
    grant execute on dbms_lock to talek;

Connect to the TALEK user and create the following objects:

    create table big_table (id integer primary key, text varchar2(300), document clob);
    create sequence big_table_id;
    alter table big_table enable row movement;

Populate the big table:

    begin
      for i in 1..5 loop
        insert into big_table select big_table_id.nextval, 
                                    object_name, 
                                    dbms_random.string('a', dbms_random.value(1000,32000)) 
                                from all_objects;
        commit;
      end loop;
    end;
    /

Now, that we have the table, let's fragment it:

    delete from big_table where id between 1000 and 100000;
    commit;

Good! Let's also add a dummy trigger:

    create or replace trigger trg_big_table_dummy 
      before insert or update or delete
      on big_table
    begin
      raise_application_error(-20000, 'Read only!');
    end;
    /

Let's have a look on the fragmentation:

    17:03:02 SQL> exec dbms_stats.gather_table_stats(user, 'big_table', cascade => true);

    PL/SQL procedure successfully completed.

    17:04:05 SQL> select table_name,round((blocks*8),2) "size (kb)" ,
    17:04:05   2         round((num_rows*avg_row_len/1024),2) "actual_data (kb)",
    17:04:05   3         (round((blocks*8),2) - round((num_rows*avg_row_len/1024),2)) "wasted_space (kb)"
    17:04:05   4  from user_tables
    17:04:05   5  where table_name='BIG_TABLE';

    TABLE_NAME size (kb) actual_data (kb) wasted_space (kb)
    ---------- --------- ---------------- -----------------
    BIG_TABLE     113416         19472.43          93943.57

How about the LOB segment?

    17:08:51 SQL> select bytes/1024/1024 lob_size
    17:08:51   2    from user_segments
    17:08:51   3   where segment_name = (select segment_name from user_lobs where table_name='BIG_TABLE');

    LOB_SIZE
    --------
        2112

    17:42:27 SQL> select sum(length(document))/1024/1024 real_lob_size from big_table;

    REAL_LOB_SIZE
    -------------
      197.500557

Perfect! Let's simulate a hot storm of short locks on the target table:

    17:29:51 SQL> declare
    17:29:51   2    l_min integer;
    17:29:51   3    l_max integer;
    17:29:51   4  begin
    17:29:51   5    select min(id), max(id)
    17:29:51   6      into l_min, l_max
    17:29:51   7      from big_table;
    17:29:51   8    loop
    17:29:51   9      -- forever
    17:29:51  10      for l_rec in (select 1 from big_table 
                                     where id = dbms_random.value(l_min, l_max) 
                                     for update) loop
    17:29:51  11        commit;
    17:29:51  12      end loop;
    17:29:51  13      dbms_lock.sleep(0.1);
    17:29:51  14    end loop;
    17:29:51  15  end;
    17:29:51  16  /

Shrinking Process
-----------------

From another session start the shrinking process:

    17:44:55 SQL> alter table big_table shrink space compact cascade;

    Table altered.

    Elapsed: 00:02:56.33

Let's try to also move the HWM:

    17:38:40 SQL> alter table big_table shrink space cascade;

    ... is just waiting forever ...

It's obvious! We're not going to be able to move the HWM as long as we have active transactions on that table. So,
we'll stop the storm session and try again. This time the HWM move will succeed:

    17:51:30 SQL> alter table big_table shrink space cascade;

    Table altered.

    Elapsed: 00:00:28.46

Interesting enough, the HWM move wasn't a lightning fast operation. It took 28s on a pretty small table. Anyway, let's see the 
fragmentation after shrinking:

    TABLE_NAME size (kb) actual_data (kb) wasted_space (kb)
    ---------- --------- ---------------- -----------------
    BIG_TABLE      22544         19472.43           3071.57

    LOB_SIZE
    --------
    386.5625

Great! It worked! How about the indexes status after shrink?

    18:05:04 SQL> select distinct status from user_indexes where table_name='BIG_TABLE';

    STATUS
    --------
    VALID


Comparison with the MOVE TABLESPACE approach
--------------------------------------------

Let's compare the shrink process with the other approach of using the ``ALTER <table> MOVE``. We're interested in finding
the compression rate:

    18:09:40 SQL> alter table big_table move tablespace users;

    Table altered.

The compression rate is:

    TABLE_NAME size (kb) actual_data (kb) wasted_space (kb)
    ---------- --------- ---------------- -----------------
    BIG_TABLE      22264         19472.43           2791.57

    LOB_SIZE
    --------
    386.5625

Not a big difference at all.

Conclusions
-----------

1. The shrink segment process can take place, indeed, online. The only exception is during the HWM movement. 
1. During the shrink process the top wait event was: ``db sequential read``.
1. The indexes are maintained, which is great. But don't forget to use the ``CASCADE`` clause.
1. Only segments in automatic segment space management tablespaces can be shrunk using this method.
1. You can't shrink tables which have functional indexes defined on them.
1. Compressed tables are not supported.
1. There are other not supported features as well, not the same from one version to another (e.g. in 10g
   LOBs can't be shrunk).
1. It is supported on standard edition.
1. I'm not sure, but I assume the rows movement inside the segment is done in a "deferrable transaction" way, 
   otherwise would not be possible to shrink a parent table. Remember, the shrink is implemented as a bunch of
   DELETE/INSERT operations.
1. Tracing the session doing the shrink operation I wasn't able to see the DELETE/INSERT sequence of
   commands, which means that the shrink takes place deep underneath, in the Oracle kernel guts.
1. The triggers are not a problem. Only if they depend somehow on the ROWID.
1. You cannot shrink in parallel.
1. The ``v$session_longops`` view is not updated throughout this shrink process. However, you can use the following
   query as a rough estimation of the amounts of read/writes as part of the shrink process:

        select a.event, a.WAIT_TIME, c.SQL_TEXT, 
              c.PHYSICAL_READ_BYTES / 1024 / 1024 / 1024 "GB_READ", 
              c.PHYSICAL_WRITE_BYTES / 1024 / 1024 / 1024 "GB_WRITE"
        from v$session_wait a , v$session b , v$sql c
        where a.SID = <sid_of_the_shrink_session> 
        and a.sid = b.sid
        and b.SQL_ID = c.SQL_ID;

1. When you move the HWM, this counts as a DDL on the table and all the cached cursors from the library cache will be
   invalidated.
1. The HWM movement is not as fast as one might think. It happened to me once to have something like this:

        16:30:15 SQL> alter table "PCP"."ORCHESTRATION_REPORT" shrink space COMPACT cascade;
        Table altered.

        Elapsed: 00:11:16.47

        16:42:34 SQL> alter table "PCP"."ORCHESTRATION_REPORT" shrink space cascade;
        Table altered.

        Elapsed: 00:47:54.28
   Yes, the HWM movement took more than the online defrag!

1. You may choose to shrink just a LOB using: ``ALTER TABLE <table> MODIFY LOB(<lob_column>) (SHRINK SPACE)``
1. There's also a so-called segment advisor but, apparently, it needs to be licensed under the diagnostics pack since it
   heavily relies on AWR.
1. Using the shrink approach is more appealing then online reorganization because it's far simpler and there's no
   need for additional storage. Anyway, as I said, online reorganization is not available on standard edition.

References
----------

Some interesting Oracle support notes:

  * [Doc ID 242090.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=242090.1)
  * [Doc ID 820043.1](https://support.oracle.com/epmos/faces/DocumentDisplay?id=820043.1)
