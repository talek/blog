---
layout: post
Title: Response Files
---

One of the frequent tasks a DBA must carry out is to install Oracle software on
new hosts, along with, presumably, new Oracle databases. Usually, it's just a
matter of launching the Oracle installer or "dbca" utility and then following
the wizard. Not a big deal! However, this approach has two inconveniences:

  * you need a X server in order to use the GUI
  * the lack of standardization: it's hard to remember all the options,
    locations and so on, not to mention it's error prone
  * if many installations are to be done, it's not funny to click-click through
    all those wizard pages

Oracle addresses these kind of problems by means of:

  * silent installations
  * response files

Response Files Location
=======================

As an example, we're going to install a new Oracle 12c using the silent/response
files facility. We assume that the server is already prepared for such an
installation, so all prerequisites have been already taken care of.

The first step is to install the Oracle software. After we downloaded the
required archive and unzip it, on Linux, we end up with something like this:

    [oracle@mercur 12102_linux64]$ find . -mindepth 2 -maxdepth 2 -printf '%M %u %p\n'
    drwxrwx--- root ./database/rpm
    drwxrwx--- root ./database/stage
    -rwxrwx--- root ./database/runInstaller
    drwxrwx--- root ./database/sshsetup
    -rwxrwx--- root ./database/readme.html
    -rwxrwx--- root ./database/welcome.html
    drwxrwx--- root ./database/install
    drwxrwx--- root ./database/response

The template response files are in the "database/response" folder:

    [oracle@mercur 12102_linux64]$ ls -al database/response/
    total 120
    drwxrwx--- 1 root vboxsf  4096 Jul  7  2014 .
    drwxrwx--- 1 root vboxsf  4096 Jan 19 12:02 ..
    -rwxrwx--- 1 root vboxsf 74822 Apr  4  2014 dbca.rsp
    -rwxrwx--- 1 root vboxsf 25036 Jul  7  2014 db_install.rsp
    -rwxrwx--- 1 root vboxsf  6038 Jan 24  2014 netca.rsp

So, there's a template for the software installation, one for the network setup
and one for the database installation using "dbca".

Create and Use the Response File
================================

If a RSP file is already provided, the simplest method is to make a copy of it
and edit it so that to be customized to the DBA/organization standard.

The other option is to record a RSP file using the installer if, of course, such
an option exists for that installer. For example, up to 11.2, the Oracle
Universal Installer used to have the "-record" parameter which might have be
used in order to generate a response file. Interestingly enough, this option is
not a valid one anymore in 12c.

As soon as you have the response file, you may start the installation:

    [oracle@mercur database]$ ./runInstaller -silent -responseFile /media/sf_bazar/orakits/rsps/db_install_12102_ee_norac.rsp -waitforcompletion
    Starting Oracle Universal Installer...

    Checking Temp space: must be greater than 500 MB.   Actual 39043 MB    Passed
    Checking swap space: must be greater than 150 MB.   Actual 4086 MB    Passed
    Preparing to launch Oracle Universal Installer from /tmp/OraInstall2015-01-19_02-12-48PM. 
    Please wait ...You can find the log of this install session at:
    /u01/app/oraInventory/logs/installActions2015-01-19_02-12-48PM.log
    The installation of Oracle Database 12c was successful.
    Please check '/u01/app/oraInventory/logs/silentInstall2015-01-19_02-12-48PM.log' for more details.

    As a root user, execute the following script(s):
            1. /u01/app/oraInventory/orainstRoot.sh
            2. /u01/app/oracle/product/12.1.0/2e_db/root.sh

    Successfully Setup Software.

As root, run the above two scripts and you're good to go.

To silently create the listener, we can also use the Oracle provided RSP file.
For a standard LISTENER, there's no need to edit anything.

    [oracle@mercur response]$ /u01/app/oracle/product/12.1.0/2e_db/bin/netca -silent -responsefile /media/sf_bazar/orakits/12102_linux64/database/response/netca.rsp 

    Parsing command line arguments:
        Parameter "silent" = true
        Parameter "responsefile" = /media/sf_bazar/orakits/12102_linux64/database/response/netca.rsp
    Done parsing command line arguments.
    Oracle Net Services Configuration:
    Profile configuration complete.
    Oracle Net Listener Startup:
        Running Listener Control: 
          /u01/app/oracle/product/12.1.0/2e_db/bin/lsnrctl start LISTENER
        Listener Control complete.
        Listener started successfully.
    Listener configuration complete.
    Oracle Net Services configuration successful. The exit code is 0

Finally, a database can be created using a response file. You can use the
following command:

    [oracle@mercur ~]$ dbca -silent -responseFile /media/sf_bazar/orakits/rsps/dbca_12c_noasm_pluggable.rsp                                                                                        
    Enter SYS user password: 
    
    Enter SYSTEM user password: 
    
    Enter PDBADMIN User Password: 
    
    Cleaning up failed steps
    4% complete
    Copying database files
    5% complete
    6% complete
    12% complete
    17% complete
    30% complete
    Creating and starting Oracle instance
    32% complete
    35% complete
    36% complete
    37% complete
    41% complete
    44% complete
    45% complete
    48% complete
    Completing Database Creation
    50% complete
    53% complete
    55% complete
    63% complete
    71% complete
    74% complete
    Creating Pluggable Databases
    79% complete
    100% complete
    Look at the log file "/u01/app/oracle/cfgtoollogs/dbca/mercdb/mercdb.log" for further details.

Mission accomplished! It is important to note that the vast majority of
installers accept a wide range of parameters which corresponds to options from
the response file. The argument value from the command line has precedence.

Common Issues
=============

Relative Paths
--------------

Pay attention not to provide the response file with a relative location.
Apparently, OUI is not smart enough to figure out how to handle this scenario:

    [oracle@mercur database]$ ./runInstaller -silent -responseFile ../../rsps/db_install_12102_ee_norac.rsp
    Starting Oracle Universal Installer...

    Checking Temp space: must be greater than 500 MB.   Actual 39052 MB    Passed
    Checking swap space: must be greater than 150 MB.   Actual 4086 MB    Passed
    Preparing to launch Oracle Universal Installer from /tmp/OraInstall2015-01-19_01-06-47PM. Please wait ...[oracle@mercur database]$ [FATAL] [INS-10101] The given response file ../../rsps/db_install_12102_ee_norac.rsp is not found.
      CAUSE: The given response file is either not accessible or do not exist.
      ACTION: Give a correct response file location. (Note: relative path is not supported)
    A log of this session is currently saved as: /tmp/OraInstall2015-01-19_01-06-47PM/installActions2015-01-19_01-06-47PM.log. Oracle recommends that if you want to keep this log, you should move it from the temporary location.

Background Installation
-----------------------

By default, the OUI is launching the installation session in background as a
separate process. Personally, I don't like this style as it force me to
immediately go and tail the log file. Furthermore, in Linux, the console is not
completely detached and random messages are printed by the background install
process. In order to avoid this, you must provide the ``-waitforcompletion``
parameter. The problem I've noticed however when using this parameter is that
the log file is automatically deleted at the end of the installer command. Yet,
another stupidity of Oracle.

OUI Complains about My Oracle Support
-------------------------------------

If you get:

    [oracle@mercur database]$ ./runInstaller -silent -responseFile /media/sf_bazar/orakits/rsps/db_install_12102_ee_norac.rsp -waitforcompletion
    Starting Oracle Universal Installer...

    Checking Temp space: must be greater than 500 MB.   Actual 39052 MB    Passed
    Checking swap space: must be greater than 150 MB.   Actual 4086 MB    Passed
    Preparing to launch Oracle Universal Installer from /tmp/OraInstall2015-01-19_01-27-51PM. Please wait ...
    [WARNING] - My Oracle Support Username/Email Address Not Specified
    A log of this session is currently saved as: /tmp/OraInstall2015-01-19_01-27-51PM/installActions2015-01-19_01-27-51PM.log. 
    Oracle recommends that if you want to keep this log, you should move
    it from the temporary location.

Check if the ``DECLINE_SECURITY_UPDATES`` variable is set to ``true`` into the
response file.

Things I don't Like
===================

Below are the things I don't like as far as the response files feature is
concerned:

  * The response file is a static dumb file. You cannot take values from
    environment variables or interact with other external interfaces. However,
    I think you can use ``m4`` parser to generate a new response file on the
    fly.
  * The character-sets from the provided ``dbca.rsp`` template file are kind of
    stupid. For example, for the default characterset, the  is US7ASCII
    assumed which isn't such a good idea.
  * I couldn't find anything related to ``OBFUSCATEDPASSWORDS`` parameter. How
    is it supposed to obfuscate these passwords? You may find some clues if you
    opt to install the database using OUI and then if you're take a look into
    the log. You may find something like this:

        INFO: Configuration assistant "Oracle Net Configuration Assistant" succeeded 
        INFO: Command = /export/home/oracle/product/12.1.0/Db_3/bin/dbca  
                -progress_only 
                -createDatabase 
                -templateName General_Purpose.dbc 
                -gdbName orcl -sid orcl  
                -sysPassword 052b3cccab83184dd1ba5e728aa1ff1e189a90088c2da70291  
                -systemPassword 053e5dc6ab23642c16821c915b858ddc23d87c4a357e140d8e  
                -sysmanPassword 05b123a3abe32968b37230926a9bfc13f4d4ef5e52f01e095d  
                -dbsnmpPassword 058be1aeaba392a9299e7d3d8215720402fe045d76a03d4506  
                -emConfiguration LOCAL     
                -datafileJarLocation /export/home/oracle/product/12.1.0/Db_3/assistants/dbca/templates  
                -datafileDestination /export/home/oracle/product/12.1.0/oradata/ 
                -responseFile NO_VALUE   
                -characterset WE8ISO8859P1   
                -obfuscatedPasswords true

    How those passwords are encrypted I have no clue. As an workaround you can
    get those values from the log and use it in your own response file. Of
    course, this is cumbersome and something you don't want to do too often.
  * You can't specify the options you want to install directly into the response
    file. There's nothing related to "Spacial" option or "Apex" there. The only
    way is to create your own custom template and use it in the response file.
