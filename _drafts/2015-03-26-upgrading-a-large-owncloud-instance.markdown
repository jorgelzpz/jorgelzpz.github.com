---
layout: post
title:  "Upgrading a large ownCloud instance"
date:   2015-03-26 19:17
---

These last weeks I had to upgrade a large [ownCloud](http://www.owncloud.org)
installation. It was the community edition, so there was little to none
information about how to upgrade such big instances.  Many ownCloud community
installations seem to be running on small organizations, so most recommendations
about upgrading were not valid for this instance. I had to upgrade **3 major
releases**, from ownCloud 4.5 to ownCloud 7, so the estimated time for this was
of several days. Crazy, isn't it?

This ownCloud instance uses MySQL 5.5 as the database manager.

How large was this instance? Let's see some numbers:

* 6 TB of information stored
* Almost 4 million files managed
* More than 11.000 active users

Using the [standard update
method](https://doc.owncloud.org/server/5.0/admin_manual/maintenance/update.html)
on a testing environment showed a terrible slow speed when updating the database
schema. After some calculations, seemed that the upgrade would take more than 24
hours to complete, supposing there were no errors during the update. Remember
that I had to upgrade two more times, so the service would be unavailable for
several days.

After some quick searches on Google, I found out that this is a common issue
when upgrading ownCloud. The database has to be converted row by row, generating
a lot of subsequent read and write queries.

On a relatively small installation this could take just some minutes, but as the
number of files increases, the required time is way too much. How can we
overcome this issue?

Fortunately, this instance met some conditions that would make the process
shorter:

