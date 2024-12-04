---
layout: post
title: "Replacing dhcpcd with dhcp6leased in OpenBSD"
date: 2024-10-12
---

I have been using OpenBSD as a router at home for a long time. There are [many reasons to choose OpenBSD for this task](https://openbsdrouterguide.net/#why-openbsd), being one of them that the base install includes everything you need.

If your Internet provider has IPv6 support there is a chance it uses [DHCPv6-PD](https://en.wikipedia.org/wiki/Prefix_delegation) to delegate your network a prefix. In the past, if you wanted to get an IPv6 prefix in OpenBSD you had to use a third party package like [dhcpcd](https://roy.marples.name/projects/dhcpcd). Not to be confused with [dhcpd(8)](https://man.openbsd.org/dhcpd.8), note the additional `c` in the former.

It works fine, but it is not part of the base installation. It is not a very big issue, but it is considered good practice sticking with the base system for simplicity and security reasons.

Fortunately [OpenBSD 7.6](https://www.openbsd.org/76.html) ships with a new daemon in the base system called [dhcp6leased(8)](https://man.openbsd.org/dhcp6leased.8), which can be easily used to replace `dhcpcd`. Let me show you how I did it.

### dhcpcd configuration

My `dhcpcd` configuration looked like this:

```
allowinterfaces pppoe0
ipv6only
nooption domain_name_servers
nooption domain_name
duid
persistent
option rapid_commit
option interface_mtu
require dhcp_server_identifier
slaac private
waitip 6

nohook resolv.conf, yp, hostname, ntp

interface pppoe0
        ipv6rs
        ia_na 1
        # request /56 prefix from the provider to use for downstream networks
        ia_pd 2/::/56 igc3/0
```

It is the result of reading multiple guides and articles about `dhcpcd` and testing it once and again. My provider uses PPPoE, so testing DHCPv6-PD required re-restablishing the connection on every change.

The most important part of the configuration is the `interface` section. My Internet provider ([DIGI](https://www.digimobil.es/)) assigns every customer a `/56` IPv6 prefix which I propagate to the `igc3` interface. This interface is connected the rest of my local network, and [rad(8)](https://man.openbsd.org/rad.8) automatically advertises it to the rest of the clients connected to the network.

### Upgrading to OpenBSD 7.6

The upgrade process in OpenBSD was very quick and painless, as usual. It only took around 5 minutes, and required no configuration changes at all.

### Configuring dhcp6leased

After taking a look at the [dhcp6leased.conf(5)](https://man.openbsd.org/dhcp6leased.conf.5) man page, I was able to quickly write the following configuration file:

```
request prefix delegation on pppoe0 for { igc3/64 }
```

**Updated 2024-12-04**: I noticed SLAAC was not working as expected in my machines. They were configuring themselves using a prefix that did not match the one the firewall was getting from the PPPoE connection. In the end, I found that SLAAC does not work with prefixes that are not `/64`, and also that [Amazon Echo devices send Router Advertisements](https://mastodon.social/@jorgelzpz/113448507526157269). I have updated the configuration file to use a `/64` prefix, even if my ISP assigns a `/56`.

I commented out all lines related to `dhcpcd` in the `/etc/rc.conf.local` file, and then enabled `dhcp6leased`:

```
# rcctl enable dhcp6leased
```

One reboot later, the IPv6 prefix was being correctly assigned and everything was working as expected:

```
Oct 11 19:54:16 firewall dhcp6leased[15122]: Soliciting lease on pppoe0
Oct 11 19:54:16 firewall dhcp6leased[15122]: Requesting lease on pppoe0
Oct 11 19:54:16 firewall dhcp6leased[64076]: prefix delegation #0 2a0c:5a81:[...]:4c00::/56 received on pppoe0 from server 0 0020000863800000080b7aee13e7b1eb1bc751302be1f8b8a
```

I was able to remove `dhcpcd` from the system and call it a day:

```
# pkg_delete dhcpcd
# /usr/sbin/userdel _dhcpcd
# /usr/sbin/groupdel _dhcpcd
```


### Configuring rad

*Added on 2024-12-04*

[rad(8)](https://man.openbsd.org/rad.8) is tasked with sending Router Advertisements in OpenBSD. I found that an [Amazon Echo device was sending RAs](https://mastodon.social/@jorgelzpz/113448507526157269), messing with the prefix my ISP assigned.

I solved it by setting the router preference to _High_ in the `/etc/rad.conf` file:

```
interface igc3 {
        default router yes
        router preference high

        dns {
                nameserver fe80::1
                search myhomedomain.home
        }
}
```
