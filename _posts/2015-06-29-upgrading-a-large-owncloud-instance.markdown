---
layout: post
title:  "Upgrading a large ownCloud instance"
date:   2015-06-29 20:49
---

These last weeks I had to upgrade a large [ownCloud](http://www.owncloud.org)
installation. It was the community edition, so there was little to none
information about how to upgrade such big instances.  Many ownCloud community
installations seem to be running on small organizations, so most recommendations
about upgrading were not valid for this instance. I had to upgrade **3 major
releases**, from ownCloud 4.5 to ownCloud 7.

This ownCloud instance used MySQL 5.5 as the database manager.

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
to be updated row by row, generating a lot of subsequent read and write
queries.

On a relatively small installation this could take just some minutes, but as the
number of files increases, the required time to complete is way too much.

Fortunately, this instance met two conditions that made me able to shorten the
upgrade process:

* It didn't use encryption (important)
* Was using user names (``uid``) as internal identifiers inside the
  data directory

So I came up with an alternate ownCloud upgrade process.

Does this method actually improve the upgrade time? Absolutely! It only took
about __2 hours__ to upgrade the instance described above (~4.000.000 files). And
how can this be possible? There is an easy answer to this question: given that
ownCloud takes a very long time when upgrading a populated database, why not work
on a clean database and restore everything at the end?

Let's do it!

Site shutdown
-------------

Your users should not be using ownCloud while you are in the middle of the
upgrade. Deny public access to your instance using your firewall or your frontend
web servers.

Backup
------

Before proceeding to upgrade make sure to backup the ownCloud directory and the
``data`` directory, and dump the database. In case you use MySQL I suggest using
``mysqldump`` like this:

{% highlight bash %}
mysqldump --default-character-set=utf8 -E --single-transaction \
  -q -n -uowncloud -p owncloud > /tmp/dump_owncloud.sql
{% endhighlight %}

Preparing the database
----------------------

Instead of working on the same ownCloud database, using a copy of the  database
gives you an easy fall-back plan, so proceed to create a new user and database:

{% highlight mysql %}
GRANT ALL PRIVILEGES ON owncloudN TO owncloudN@localhost identified by 'XXXXXXX';
CREATE DATABASE owncloudN DEFAULT CHARACTER SET utf8;
FLUSH PRIVILEGES;
{% endhighlight %}

And then import the old ownCloud data into this new database:

{% highlight console %}
mysql --default-character-set=utf8 -uowncloudN -p owncloudN < /tmp/dump_owncloud.sql
{% endhighlight %}

Note that you should use proper user/database names instead of `owncloudN`. I'd
suggest using ownCloud major version target (e.g. `owncloud8`).

ETags and shares
----------------

We are going to wipe our new database, so we need to keep a
copy of some important data. Most important data is:

* File [ETags](https://en.wikipedia.org/wiki/HTTP_ETag), so ownCloud desktop
clients recognize currently synchronized files
* User configured file/directory shares

Knowing exactly how to backup and restore this information depends on your source
and target ownCloud versions. Have a look at some [ownCloud upgrade scripts](https://github.com/adobo/owncloud_upgrade) that I have released. Contributions
are welcome!

Once you have both ETags and shares current data backed up, put them in a safe place
until the last step.

Wipe the database
-----------------

The following SQL sentences will put your ownCloud database to a mostly clean
state:

{% highlight sql %}
CREATE TABLE oc_filecache2 LIKE oc_filecache;
DROP TABLE oc_filecache;
RENAME TABLE oc_filecache2 TO oc_filecache;
CREATE TABLE oc_properties2 LIKE oc_properties;
DROP TABLE oc_properties;
RENAME TABLE oc_properties2 TO oc_properties;
CREATE TABLE oc_share2 LIKE oc_share;
DROP TABLE oc_share;
RENAME TABLE oc_share2 TO oc_share;
CREATE TABLE oc_ldap_user_mapping2 LIKE oc_ldap_user_mapping;
DROP TABLE oc_ldap_user_mapping;
RENAME TABLE oc_ldap_user_mapping2 TO oc_ldap_user_mapping;
{% endhighlight %}

Note: if you are upgrading from an old ownCloud version (e.g. 4.0.x), then `oc_filecache` table will not exist, because it was called `oc_fscache`. In that case, work with the `oc_fscache` table instead of `oc_filecache`.


Perform the ownCloud upgrade
----------------------------

Upgrade ownCloud as their documentation explains. Remember to copy your old `config.php`
file and __do not forget updating the database connection credentials to use your new
database__!

Rescan all user files
---------------------

Now it's time to let ownCloud to know about your user files. I used [GNU Parallel](https://www.gnu.org/software/parallel/) to make it faster.

Before launching the scan, get a list of current users from the data directory,
and then loop on it. Adapt the following command to your case:

{% highlight console %}
# find /owncloud-files/ -mindepth 1 -maxdepth 1 -type d | cut -f 3 -d '/' > /tmp/owncloud_users
# mkdir /tmp/owncloud_sync
# time (cat /tmp/owncloud_users | parallel --gnu -j8 --progress \
    --joblog /tmp/scan_log \
    --tmpdir /tmp/owncloud_sync/ \
    --files 'sudo -u apache -H php /var/www/html/owncloud/occ files:scan {}')
{% endhighlight %}

The command above launches 8 parallel file scan jobs. You can control its progress
by having a look at `/tmp/scan_log` and using the `ps` command.

This step will take most of the upgrade process time. Depending on your machine
specs (storage, memory and CPU) it can take more or less time. Adjust the
number of parallel processes to your system.

Restore ETags and shares
------------------------

Use the data from the step _ETags and shares_ to restore the old ownCloud state.

The [ownCloud upgrade scripts](https://github.com/adobo/owncloud_upgrade) project
can be helpful.

Et voilá! Now you are ready to go!

About upgrading multiple releases
---------------------------------

If you have to upgrade your ownCloud instance more than one major release, apply
the database cleaning process just once (in the beginning), and do the file scan
and ETags/shares restore just after upgrading to the latest ownCloud. Do not
worry about intermediate versions.
