---
layout: post
title: Stupid "root.sh" Crash when Installing GI 12.1.0.2
comments: true
---

Funny thing happen today. I was asked to install a 12.1.0.2 RAC database. The
hardware and the OS prerequisites were, in their vast majority, previously
addressed, so everything was ready for the installation. Of course, as you
already know, Oracle is quite picky about the expected environment (and for a
good reason) and it checks a lot of things before starting the actual
installation. Of course, sometimes it does this horribly. For example, take a
look at the following screenshot:

![shm prereq]({{ site.baseurl }}assets/images/shm.png "shm prereq")

Apparently this is due to unpublished `Bug 19031737` and can be ignored if the
setting is correctly set in the OS.

Everything went smooth until I ran the `root.sh` script. I ended up with:

    /u01/app/12.1.0/2_grid/bin/sqlplus -V ... failed rc=1 with message:
     Error 46 initializing SQL*Plus HTTP proxy setting has incorrect value SP2-1502: The HTTP proxy server specified by http_proxy is not accessible

    2016/03/30 13:53:15 CLSRSC-177: Failed to add (property/value):('VERSION'/'') for checkpoint 'ROOTCRS_STACK' (error code 1)

    2016/03/30 13:53:39 CLSRSC-143: Failed to create a peer profile for Oracle Cluster GPnP using 'gpnptool' (error code 1)

    2016/03/30 13:53:39 CLSRSC-150: Creation of Oracle GPnP peer profile failed for host 'ucgprdora001'

    Died at /u01/app/12.1.0/2_grid/crs/install/crsgpnp.pm line 1846.
    The command '/u01/app/12.1.0/2_grid/perl/bin/perl -I/u01/app/12.1.0/2_grid/perl/lib -I/u01/app/12.1.0/2_grid/crs/install /u01/app/12.1.0/2_grid/crs/install/rootcrs.pl ' execution failed

Stupid! Our sysadmin set a global `http_proxy` variable which confused `sqlplus`
and, by extension, the `root.sh` script.

Wouldn't be a good idea `cluvfy` to check also if this variable is correctly set
(or unset)?

Until then, I put this as a note to myself: don't forget to always unset
`http_proxy` variable before starting to install a new Oracle server.
