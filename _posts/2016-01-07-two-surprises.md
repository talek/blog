---
layout: post
title: Two Surprises on Upgrading Oracle to 12.1.0.2
comments: true
---

Yesterday we migrated one of the customers Oracle servers from 11.2 to 12.1.0.2.
It was a single instance server, the volume of data wasn't very impressive and
it was also a Standard Edition installation. So, nothing special, nothing scary.
We though it would be a simple task. Hmmm... Let's see.

## What Oracle Edition?

Licensing, editions, options... I'm starting to get annoyed by all these
non-technical topics. But, that's life, we have to deal with them. In this
particular case, the customer asked us to upgrade their standard edition
database running on 11.2.0.4 to Oracle 12.1.0.2. Super, but guess what? There's
no plain Standard Edition or Standard Edition One released for 12.1.0.2, just
for 12.1.0.1. The only available edition on 12.1.0.2 is Oracle SE2 (Standard
Edition 2) and, apparently, that's the only standard edition Oracle is committed
to support for the next releases. For details, please see [Note:
2027072.1](https://support.oracle.com/epmos/faces/DocumentDisplay?parent=DOCUMENT&sourceId=1905806.1&id=2027072.1).

## DATA\_PUMP\_DIR Bug

The upgrade was a success, at least this is what we thought. Well, bad luck. In
the morning the customer reported that a schema which is supposed to be
replicated every night was not there. Ups! We analysed the logs and we could
notice that the `impdp` utility was complaining that the source dump couldn't be
accessed. What? The import was configured to use the `DATA_PUMP_DIR` oracle
directory which was configured to point out to a specific location on the
database server. After the upgrade it was changed to `$ORACLE_HOME/rdbms/log`.
Great! I wasn't aware of this. Most likely, we hit the following bug:

    Bug 9006105 - DATA_PUMP_DIR is not remapped after database upgrade [ID 9006105.8]

It's an unpublished bug, so I can't see much besides what is already shown in
the subject. However, in the note they claim that they managed to fix it in
11.2.0.2 and 12.1.0.1. Well, apparently not.

## Conclusion

As always, Oracle upgrades are such a joy. Expect the unexpected... Anyways, I'm
starting to have doubts that using the `DATA_PUMP_DIR` directory is such a good
idea. In the light of what happen, I think it's always better to create and use
your own oracle directory. Not to mention, `DATA_PUMP_DIR` can't be used from
within a pluggable database if you plan to use this option on a 12c
database.
