---
layout: post
title: "My libvirt Configuration with NetworkManager"
date: "2025-10-24"
tags: [libvirt, network, NetworkManager]
description:

  I often test my work in a VM on my laptop, and I have set it up in a
  particular way that I find convenient. I wrote this post for myself
  when I need to configure another system. You can still use it as an
  inspiration.
---

Before you start telling me, I am aware of the `register='yes'`
attribute in the `<domain>` element of a libvirt network
configuration, but I prefer `dnsmasq` over `systemd-resolved`. Also,
my configuration predates libvirt v10.1.0.

## Environment

My network is managed by NetworkManager, because it can configure
wired, wireless and even Bluetooth connections without much hassle,
and it is nicely integrated with KDE.

I set up a NAT network for my VMs, so they can access the Internet
through a wired (when docked at home) or wireless (when on the go)
connection, and can be accessed locally even when offline.

## Libvirt Configuration

My NAT network is a simple IPv4 network in the private range
192.168.86.0/24 (250 VMs should be enough for everyone, right?). The
number 86 is arbitrary, but it is also the ASCII code for ‘V’ (as
in “VM network”). My libvirt configuration is:

```
<network>
  <name>default</name>
  <forward mode='nat'/>
  <bridge name='virbr0' stp='on' delay='0'/>
  <domain name='virt.tesarici.cz' localOnly='yes'/>
  <dns>
    <host ip='192.168.86.1'>
      <hostname>gate</hostname>
    </host>
  </dns>
  <ip address='192.168.86.1' netmask='255.255.255.0' localPtr='yes'>
    <dhcp>
      <range start='192.168.86.2' end='192.168.86.254'/>
    </dhcp>
  </ip>
</network>
```

The gateway IP address is 192.168.86.1, all other addresses are
dynamically assigned to VM hosts via DHCP. The `hostname` tag adds a
DNS entry for the gateway.

Note the `localOnly` and `localPtr` attributes. Since the network is
private, we don't want to forward queries to an external DNS server
(and it could even loop, see below).

## Exposing it to the Host

The above settings work nicely from a VM, but I can't use a guest
hostname from the host, which is suboptimal.

So, first, let's also use dnsmasq in the host. To do that, save this
as `/etc/NetworkManager/conf.d/use-dnsmasq.conf`:

```
[main]
dns=dnsmasq
```

Next, let's update the dnsmasq configuration files when the libvirt
network is started or destroyed. I don't like duplicating
configuration, so I added this hook script as
`/etc/libvirt/hooks/network.d/nm-dnsmasq`:

```
#! /usr/bin/env python3

# Requires:
#   python3-lxml
#   python3-dnspython

from sys import argv, stdin

def cfgpath(ifname):
    return '/etc/NetworkManager/dnsmasq.d/libvirt-' + ifname + '.conf'

def restart_dnsmasq():
    from os import spawnlp, P_NOWAIT
    spawnlp(P_NOWAIT, 'nmcli', 'nmcli', 'general', 'reload', 'dns-full')
    return 0

if argv[2] == 'started':
    cfg = open(cfgpath(argv[1]), 'w')
    from lxml import etree
    tree = etree.parse(stdin)
    ips = tree.xpath('/hookData/network/ip')
    for domain in tree.xpath('/hookData/network/domain'):
        dom = domain.get('name')
        for ip in ips:
            addr = ip.get('address')
            print(f'server=/{dom}/{addr}', file=cfg)

    from ipaddress import ip_network
    from dns import reversename
    for ip in ips:
        addr = ip.get('address')
        mask = ip.get('prefix') or ip.get('netmask')
        net = ip_network(f'{addr}/{mask}', False)
        prefix = net.prefixlen
        if net.version == 4:
            prefix += 7 - (prefix - 1) % 8
            revdepth = (net.max_prefixlen - prefix) // 8
        else:
            prefix += 3 - (prefix - 1) % 4
            revdepth = (net.max_prefixlen - prefix) // 4
        for subnet in net.subnets(new_prefix=prefix):
            rev = reversename.from_address(str(subnet.network_address))
            dom = rev.split(len(rev) - revdepth)[1].to_text()
            print(f'server=/{dom}/{addr}', file=cfg)
    cfg.close()
    restart_dnsmasq()

elif argv[2] == 'stopped':
    from os import unlink
    unlink(cfgpath(argv[1]))
    restart_dnsmasq()
```

This hook will manage libvirt-*.conf files under
`/etc/NetworkManager/dnsmasq.d/` necessary to forward DNS queries to
the libvirt dnsmasq as appropriate. There is a bit of non-trivial
logic to create configuration for reverse DNS if the network prefix is
not a multiple of 8 for IPv4 or a multiple of 4 for IPv6. However,
libvirt may struggle with `localPtr=yes` in that case.

Last, add this domain to the search list, so I can type `ssh dmafix`
instead of `ssh dmafix.virt.tesarici.cz`. Save this as
`/etc/NetworkManager/conf.d/search-virt-domain.conf`:

```
[global-dns]
searches=virt.tesarici.cz
```

Reload NetworkManager (`systemctl reload NetworkManager`) and start
the libvirt network.

Assuming the dmafix VM hostname is set appropriately (`hostnamectl
hostname --static dmafix`), it works great:

```
petr@mordecai:~/src/linux> ping dmafix
PING dmafix.virt.tesarici.cz (192.168.122.193) 56(84) bytes of data.
64 bytes from dmafix.virt.tesarici.cz (192.168.122.193): icmp_seq=1 ttl=64 time=0.351 ms
64 bytes from dmafix.virt.tesarici.cz (192.168.122.193): icmp_seq=2 ttl=64 time=0.448 ms
^C
--- dmafix.virt.tesarici.cz ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1028ms
rtt min/avg/max/mdev = 0.351/0.399/0.448/0.048 ms
```
