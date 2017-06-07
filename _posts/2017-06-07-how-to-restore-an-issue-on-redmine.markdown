---
layout: post
title:  "How to restore an issue on Redmine"
date:   2017-06-07 09:30
---

I take care of several [Redmine](http://www.redmine.org/) based platforms, and every now and then I
get a report about an issue that _disappeared_ under unknown circumstances. Logs show that it is
usually caused by someone with too many privileges who accidentally deleted the issue, but that's
another story.

Let's see how we can restore an accidentally (or not) removed issue. This guide assumes you have
both database and filesystem backups that contain the removed issue. The procedure is shown for
MySQL, but a similar approach can be used with other databases.

_Disclaimer_: I have used this method on Redmine 3.2 and 3.3 instances, but I'm not sure if it
works on older Redmine versions. Anyways, I take no responsibility for any unexpected consequences,
make sure you understand every step and double check before executing anything!

Database preparation
--------------------

Create a new database, which you will use to restore the backup into:

{% highlight SQL %}
CREATE DATABASE `restoreissue` DEFAULT CHARACTER SET 'utf8';
{% endhighlight %}

And then import your latest database dump inside it:

{% highlight shell %}
$ cat /path/to/dump.sql | mysql -uroot -p restoreissue
{% endhighlight %}

Restore database contents
-------------------------

Now let's assume the following parameters:

* **xxx** is the numeric id for the removed issue.
* `redmine` is your production database

Restoring database objects can be achieved by executing the following SQL statements:

{% highlight SQL %}
SET @ISSUEID = xxx;

INSERT INTO redmine.issues 
  (SELECT *
    FROM restoreissue.issues
    WHERE id = @ISSUEID);

INSERT INTO redmine.journals 
  (SELECT *
    FROM restoreissue.journals
    WHERE journalized_id = @ISSUEID
      AND journalized_type = 'Issue');

INSERT INTO redmine.journal_details
  (SELECT d.*
    FROM restoreissue.journals j 
    INNER JOIN restoreissue.journal_details d
    ON j.id = d.journal_id
    WHERE j.journalized_id = @ISSUEID
      AND j.journalized_type = 'Issue');

INSERT INTO redmine.attachments
  (SELECT * FROM restoreissue.attachments
    WHERE container_id = @ISSUEID
      AND container_type = 'Issue');
{% endhighlight %}

After running these statements, your issue should be accessible again using a browser.

Restore attachments
-------------------

This step is only required if your issue had files attached.

Let's assume your Redmine instance is located at `/opt/redmine`. Restore `/opt/redmine/files` from
your backup somewhere else, and copy your attachments to their original location adapting the
following commands to your case (e.g. by using `scp` from a snapshot of your VM instead of `cp`):

{% highlight shell %}
cd /your/backup/path
mysql -uroot -p -N -B -e \
  "SET @ISSUEID = xxx;
   SELECT CONCAT(disk_directory, '/', disk_filename)
   FROM restoreissue.attachments
   WHERE container_id = @ISSUEID
   AND container_type = 'Issue'" | while read F; do

   cp -a "$F" /opt/redmine/files/${F}
 done
{% endhighlight %}

Clean up
--------

Do not forget to remove your temporary database and your restored files.
