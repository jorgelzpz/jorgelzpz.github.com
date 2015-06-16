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
releases**, from ownCloud 4.5 to ownCloud 7.

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
several days. Crazy, isn't it?

I found out this to be a common issue when upgrading ownCloud. The database has
to be converted row by row, generating a lot of subsequent read and write
queries.

On a relatively small installation this could take just some minutes, but as the
number of files increases, required time to complete is way too much.

Fortunately, this instance met two conditions that made me able to shorten the
upgrade process:

* It didn't use encryption (this is the important)
* Was using user names (``uid``) as internal identifiers inside the
  data directory

Let me show you how to upgrade a large ownCloud instance.

Site shutdown
-------------

Your users should not be using ownCloud while you are in the middle of the
upgrade. Stop them accesing your instance using your firewall or your frontend
web servers, this is what you should do first.

Backup
------

Before proceeding to upgrade make sure to backup the ownCloud directory and the
``data`` directory, and dump the database. In case you use MySQL I suggest using
``mysqldump`` like this:

{% highlight bash %}
mysqldump --default-character-set=utf8 -E --single-transaction \
  -q -n -uowncloud -p owncloud > /tmp/dump_owncloud.sql
{% endhighlight %}

Instead of working on the same ownCloud database, I opted for working on a copy
to have an easy fall-back plan, so I created a new user and imported the
ownCloud data into the new one:

{% highlight mysql %}
GRANT ALL PRIVILEGES ON owncloud8 TO owncloud8@localhost identified by 'XXXXXXX';
CREATE DATABASE owncloud8 DEFAULT CHARACTER SET utf8;
FLUSH PRIVILEGES;
{% endhighlight %}

{% highlight console %}
# mysql --default-character-set=utf8 
  -uowncloud8 -p owncloud8 < /tmp/dump_owncloud.sql
{% endhighlight %}
