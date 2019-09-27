---
layout: post
title:  "Honoring DNS servers when connecting to a VPN on Ubuntu"
date:   2019-09-27 18:30
---

I do not certainly hate systemd, I see its power, but in my opinion some of its ideas are not
helping at all, or maybe they were not executed the best way possible.

[systemd-resolved](https://www.freedesktop.org/software/systemd/man/systemd-resolved.service.html)
is a systemd service that handles DNS resolution for local processes, and
if you combine it with an OpenVPN based profile on NetworkManager it is very likely you will run
into very weird issues with DNS.

In the case your VPN is setting a custom DNS server you might probably be experiencing an unexpected
behaviour, especially when resolving internal records: some might work, some not. This can lead to a
[DNS leak](https://en.m.wikipedia.org/wiki/DNS_leak) if you are using the VPN for privacy reasons,
but it is also very frustrating if you are trying to use your company's DNS servers to resolve
records on a split-horizon DNS setup.

Almost all modern Linux distributions use this combination of systemd and NetworkManager, including
Ubuntu, so let me show how I _fixed_ it.

After some research, seems that there are two reasons for this behaviour to happen:

1. `systemd-resolved` cached a record and will not try to resolve it again using the new DNS server
   provided by your VPN
2. NetworkManager is not using the new DNS server because it failed to resolve some record or just
   because your ISP's DNS server has higher priority

Although some people decide to completely disable `systemd-resolved`, I found a more convenient way
to keep it running and making it behave the way I want.

First of all, I disabled `systemd-resolved` caching. It can be done by setting `Cache` to `No` on
`/etc/systemd/resolved.conf`:

{% highlight ini %}
[Resolve]
# ...
Cache=no
# ...
{% endhighlight %}

And then restart `systemd-resolved`:

{% highlight shell %}
$ sudo systemctl restart systemd-resolved
{% endhighlight %}

Now we will tell NetworkManager to use `systemd-resolved` when enabling OpenVPN profiles. This can
be achieved by installing the `openvpn-systemd-resolved` package:

{% highlight shell %}
$ sudo apt-get install openvpn-systemd-resolved
{% endhighlight %}

I think this does not have effect if you do not configure your OpenVPN profile to especifically call
this tool after connecting, but I did not want to waste more time trying what happens if you remove
it.

Next step will consist in telling NetworkManager it should prioritize the DNS server from the VPN.
Assuming your VPN connection inside NetworkManager is named `MyVPN1`, run the command above:

{% highlight shell %}
$ sudo nmcli c modify MyVPN1 ipv4.dns-priority -42
{% endhighlight %}

Restarting NetworkManager and reconnecting to the VPN should do the trick.
