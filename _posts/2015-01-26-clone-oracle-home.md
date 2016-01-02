---
layout: post
title: Cloning an Oracle Home
comments: true
---

Cloning an Oracle home is required from time to time. The big advantage of
using this approach is that it preserve the patches you might have been
installed in that home. Of course, cloning is valid only for the same platform.
Don't expect to take a Windows Oracle home and clone it on a Linux box.

First step: backup the current Oracle home
==========================================

It doesn't matter what tool you use, but it's important to pay attention to the
files ownership. That tool needs to preserve that information. So, on Linux, we
can use this:

    cd $ORACLE_HOME
    tar cf - * | gzip > /tmp/oracle_home.tar.gz

In $ORACLE_HOME is the home you want to clone, the source.

Second step: restore on the target
==================================

The target can be on the same host or on a different machine:

    mkdir <target_new_oracle_home>
    cd <target_new_oracle_home>
    gunzip < /tmp/oracle_home.tar.gz | tar xf -

Third step: finalize the cloning process
========================================

Now, we need to reconfigure and attach the new home in the Oracle inventory. So,
you need to do this:

    [oracle@mercur 2e_db_cloned]$ oui/bin/runInstaller \
      -clone \
      -silent ORACLE_HOME=/u01/app/oracle/product/12.1.0/2e_db_cloned \
      ORACLE_HOME_NAME="Oracle_12c_cloned" \
      ORACLE_BASE=/u01/app/oracle \
      OSDBA_GROUP=dba \
      OSPER_GROUP=dba

    Starting Oracle Universal Installer...

    Checking swap space: must be greater than 500 MB.   Actual 4061 MB    Passed
    Preparing to launch Oracle Universal Installer from /tmp/OraInstall2015-01-26_06-19-33PM. Please wait ...
    Oracle Universal Installer, Version 12.1.0.2.0 Production
    Copyright (C) 1999, 2014, Oracle. All rights reserved.

    ... output truncated ...

    Linking in progress (Monday, January 26, 2015 6:19:47 PM EET)
    .                                                                81% Done.
    Link successful

    Setup in progress (Monday, January 26, 2015 6:20:21 PM EET)
    ..........                                                      100% Done.
    Setup successful

    Saving inventory (Monday, January 26, 2015 6:20:22 PM EET)
    Saving inventory complete
    Configuration complete

    End of install phases.(Monday, January 26, 2015 6:20:44 PM EET)
    WARNING:
    The following configuration scripts need to be executed as the "root" user.
    /u01/app/oracle/product/12.1.0/2e_db_cloned/root.sh
    To execute the configuration scripts:
        1. Open a terminal window
        2. Log in as "root"
        3. Run the scripts

    The cloning of Oracle_12c_cloned was successful.
    Please check '/u01/app/oraInventory/logs/cloneActions2015-01-26_06-19-33PM.log' for more details.

Just run the root scripts and you're good to go.
