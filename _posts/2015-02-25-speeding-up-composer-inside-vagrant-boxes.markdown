---
layout: post
title:  "Speeding up Composer inside a Vagrant box"
date:   2015-02-25 19:40
---

There has been a lot written about how to speed up Symfony when running it inside a Vagrant box. In my opinion, the most
interesting one is <a href="http://by-examples.net/2014/12/09/symfony2-on-vagrant.html">Running the Symfony application
on Vagrant without NFS below 100ms</a>. The change that has the biggest possitive impact on performance consists on
moving the `vendors/` directory outside the shared folder.

However, most of the provided solutions involve modifying your `composer.json`
file or even the front controller code to load vendors from a different
directory. You will end with something like this:

{% highlight json %}
// composer.json
{
  ...
  "config": {
    "vendor-dir": "/another/path/on/fs/vendor"
  },
  ...
}
{% endhighlight %}

And your PHP code will look like this:

{% highlight php %}
<?php
// Application front controller
require_once '/another/path/on/fs/vendor/autoload.php';
{% endhighlight %}

But what if we want to deploy the same code on a different environment, such as
our production systems? We might want our application directory to be
self-contained.

Our approach to improve our development box performance will affect where the
dependencies should be installed on production, unless we apply some
modifications to our code just after deploying to change the paths.

Making the vendors path configurable without changing your code
---------------------------------------------------------------

This solution lets you keep the `vendors` directory outside your application
directory, but defaults to the current directory.

How can this be achieved without touching a single line of code when deploying?
Using **environment variables**!

Just leave the `composer.json` untouched, without a `vendor-dir` setting, and
use the following code on your front controller:

{% highlight php %}
<?php
// Application front controller
$vendor_directory = getenv('COMPOSER_VENDOR_DIR');
if ($vendor_directory === false) {
  $vendor_directory = __DIR__ . '/vendor';
}

require_once $vendor_directory . '/autoload.php';
{% endhighlight %}

If `COMPOSER_VENDOR_DIR` environment variable is not set, Composer will look
for the `vendor/` directory inside the application directory.

You will usually set it just on your Vagrant box, preferably using a
provisioning system such as <a
href="http://www.ansible.com/get-started">Ansible</a> or <a
href="https://docs.puppetlabs.com/">Puppet</a>.

Setting an environment variable on Apache is as easy as:

{% highlight apache %}
SetEnv COMPOSER_VENDOR_DIR "/var/myapp-vendor"
{% endhighlight %}

If you are using nginx+php-fpm instead, php-fpm has to be configured like this:

{% highlight php %}
env[COMPOSER_VENDOR_DIR] = "/var/myapp-vendor"
{% endhighlight %}

Don't forget to set `COMPOSER_VENDOR_DIR` also when using `composer` from CLI.

Example
-------

If you want to see a working example, have a look at the following AgenDAV
files:

* <a
  href="https://github.com/adobo/agendav/blob/develop/web/public/index.php">Front
  controller</a>
* <a
  href="https://github.com/adobo/agendav/blob/develop/ansible/playbook.yml">Ansible
  playbook to provision the Vagrant box</a>
* <a
  href="https://github.com/adobo/agendav/blob/develop/ansible/apache/agendav">Apache
  configuration file template</a>
