---
layout: post
title: Keep SYS clean, please...
comments: true
---

One of the customers handed over to us a delivery script to be executed under SYS. We took a look and, guess what? They wanted
to create some objects in the `SYS` schema. What? Focus on our WTF face... No, no! Why? We rejected the change and asked for
details. In short, their explanation was that the application needs to dynamically change a hidden parameter using the `ALTER
SESSION` command, but they also want to know the value of that parameter before the change so that to be able to restore it
to its previous value. It was something related to a particular report which was running slow without this hidden parameter. So, 
their R&D department came up with this idea of creating a view in `SYS` based on some `X$` views and granting `SELECT` rights on this
view to the application user. They said that granting rights on `X$` is not an option, which is correct. However, is it an option
to mess up the dictionary?

Why It Is a Bad Idea To Mess Up The Dictionary?
-----------------------------------------------

Good question, let's see:

  1. because it is magic, some things may not work as you might expect. Just a glimpse of this [here](https://hoopercharles.wordpress.com/2011/11/23/why-isnt-my-index-used-when-user2-executes-this-query/).
  1. because you may loose the Oracle support in case you screw up the dictionary. It's like loosing the warranty in case you start
     disassembling the engine of your car and then you complain it doesn't work as it should. I haven't found though any
     clear Oracle disclaimer on this, just the advice: "do not create object in SYS schema". They don't tell what happens if you
     break the rule.
  1. because you may have nasty surprises during catalog upgrades
  1. because you may invalidate critical catalog objects. Have a look [here](http://dirknachbar.blogspot.ro/2011/11/why-you-should-never-create-own-objects.html).
  1. because you may have problems deploying such an application to a customer who cares about its Oracle database

A Better Way
------------

Have you heard about [APEX](https://apex.oracle.com/i/)? It has an interesting behavior: it may impersonate and execute code on behalf of another user. But this
feature, not documented, is around for a long time. The key is the `DBMS_SYS_SQL` package. So, the plan is this:

  1. create an `APP_ADMIN` user:

          grant create session,
                create procedure
             to app_admin identified by <pwd>.

  1. grant `EXECUTE` privilege on `DBMS_SYS_SQL` package to `APP_ADMIN`:

          grant execute on dbms_sys_sql to app_admin;

  1. in `APP_ADMIN` create the following function:

          create or replace function get_hidden_param(param_name varchar2)
            return varchar2
          as
            l_uid number;
            l_sqltext varchar2(300) := 'select b.ksppstvl value
                                        from sys.x$ksppi a, sys.x$ksppcv b
                                        where a.indx = b.indx
                                          and a.ksppinm = :param_name';
            l_myint integer;
            l_value varchar2(100);
            l_result number;
          begin
            select user_id into l_uid from all_users where username like 'SYS';

            l_myint:=sys.dbms_sys_sql.open_cursor();
            sys.dbms_sys_sql.parse_as_user(l_myint, l_sqltext,
                                           dbms_sql.native, l_uid);
            sys.dbms_sys_sql.bind_variable(l_myint, 'param_name', param_name);
            sys.dbms_sys_sql.define_column(l_myint, 1, l_value, 100);
            l_result := sys.dbms_sys_sql.execute(l_myint);
            if sys.dbms_sql.fetch_rows(l_myint) > 0 then
              dbms_sql.column_value(l_myint, 1, l_value);
            end if;
            sys.dbms_sys_sql.close_cursor(l_myint);
            return l_value;
          end;
          /

  1. grant `EXECUTE` rights on the above function to the application user:

          grant execute on get_hidden_param to <app_user>;

  1. now, from the application user we can play around:

          18:56:20 SQL> select app_admin.get_hidden_param('_optimizer_mjc_enabled') special_param from dual;

          SPECIAL_PARAM
          -------------
          FALSE

          18:56:22 SQL> alter session set "_optimizer_mjc_enabled"=true;

          Session altered.

          18:56:24 SQL> select app_admin.get_hidden_param('_optimizer_mjc_enabled') special_param from dual;

          SPECIAL_PARAM
          -------------
          TRUE

Conclusion
----------

  1. If you can, you should rely as less as possible to any internal/hidden/undocumented features
  1. If you really have to do it, then be gentle at least with the system. Don't mess up the dictionary!
  1. The `APP_ADMIN` user is a very powerful user, now that it has the `DBMS_SYS_SQL` granted. But so is SYS and if
     you were to choose between creating objects directly in SYS and isolating sensible/powerful code in
     another highly protected user, I'd rather vote for the second approach.
  1. Because `DBMS_SYS_SQL` is used a lot in APEX and in other Oracle products I expect to stay around for a while.
     This is a legitimate concern: as soon as you create a dependency between your code and an undocumented
     Oracle feature you're at the mercy of Oracle. They may remove or change that feature without any
     warnings and then everything will break deep down in your application guts.
