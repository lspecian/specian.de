+++
title = "The IPv6 subnets that wouldn't stop multiplying"
description = "MAAS kept creating IPv6 subnets nobody asked for. The cause wasn't MAAS discovery but my ISP rotating the delegated prefix, and the first fix didn't survive contact with netplan."
date = 2026-11-07
draft = true

[taxonomies]
tags = ["homelab", "maas", "ipv6", "networking", "incident"]

[extra]
toc = true
+++

My homelab runs [MAAS](https://maas.io/) on a small VM. It provisions the physical servers: three Dell R630s with Tesla P4s. MAAS owns PXE boot, DHCP on the provisioning VLAN, and the inventory of every fabric, VLAN and subnet it knows about.

One day I opened the subnets page and found dozens of IPv6 `/64`s on the default fabric that I had never created. I deleted thirty. A few weeks later there were forty-five more. Later still, the region controller started crashing.

This post covers what was going on, the explanation that looked right but wasn't, and why the first fix held only until the next time the network stack reinitialized.

<!-- more -->

## The symptom

Every phantom subnet looked the same: a `/64` from Deutsche Telekom's `2003:…` range, attached to `fabric-0`. That's the fabric holding the VM's uplink to my home LAN. None of them had machines, reserved ranges or a purpose. They just piled up.

## The obvious suspect, and why it was wrong

MAAS has a *network discovery* feature that listens on its interfaces and records the networks it observes. When subnets you didn't create keep appearing, discovery is the first thing to blame.

It was already off:

```console
$ maas admin maas get-config name=network_discovery
"disabled"
```

Disabling it again changed nothing, and subnets kept appearing anyway. Something else was creating them.

## What was actually happening

Three ordinary behaviours combined into one bad outcome:

1. **Telekom rotates the delegated IPv6 prefix.** Residential lines get a prefix that changes periodically. The home router then advertises the new `/64` on the LAN through router advertisements (RAs).
2. **The MAAS VM accepted those RAs.** Its LAN interface, `enp5s0`, ran with Linux defaults: `accept_ra=1`, `autoconf=1`. On every rotation it used SLAAC to configure a new global address from the new prefix.
3. **MAAS records the subnets its controllers report.** When the rack controller registers or refreshes its interfaces, it sends the region every address on every interface. If an address falls in a CIDR the region doesn't know, the region **creates a subnet for it**. This happens during interface registration, not discovery, which is why the discovery toggle had no effect.

Nothing ever deletes those subnets. Each prefix rotation added one more `/64` to the database, permanently.

## Fix #1: stop the VM from learning the prefix

The cause sat in the kernel, not in MAAS, so the fix went there too. That NIC has no reason to hold IPv6 on my LAN, so I switched RA processing, SLAAC and IPv6 itself off:

```ini
# /etc/sysctl.d/99-disable-ipv6-ra.conf
net.ipv6.conf.enp5s0.accept_ra = 0
net.ipv6.conf.enp5s0.autoconf = 0
net.ipv6.conf.enp5s0.disable_ipv6 = 1
```

I also baked it into the VM's cloud-init so a rebuild would come up clean, set `network_discovery=disabled` explicitly after `maas init`, and cleaned out the stale subnets.

The pile-up stopped, and I considered it done.

## The relapse

Months later the problem came back in a worse form. MAAS's region controller, `regiond`, began crashing.

The subnet table had grown into the **hundreds**. Some region–rack exchanges carry the full interface and subnet picture, and that payload outgrew a hard limit: MAAS uses Twisted's [AMP](https://docs.twisted.org/en/stable/core/howto/amp.html) protocol between controllers, and AMP caps a single value at 64 KB. Past that point the message can't be sent, and the region fell over.

So why was the pile-up back? `/etc/sysctl.d` is applied at boot, and only at boot. If anything reinitializes the interface's IPv6 configuration afterwards, the knobs return to their defaults. A `netplan apply` or replay does this, and so can a link flap. MAAS itself triggers netplan replays during region and rack bring-up. Once the knobs were reset, the next prefix rotation brought SLAAC straight back.

What I'd configured was a boot-time setting, not something enforced at runtime.

## Fix #2: re-assert at runtime

The durable version does the same work continuously. A oneshot service re-applies the sysctl file and flushes any global IPv6 address that has already arrived:

```sh
#!/bin/sh
# /usr/local/sbin/disable-ipv6-enp5s0.sh
set -eu
sysctl -p /etc/sysctl.d/99-disable-ipv6-ra.conf >/dev/null
ip -6 addr flush dev enp5s0 scope global 2>/dev/null || true
```

A timer runs it shortly after boot and every 15 minutes after that:

```ini
# /etc/systemd/system/disable-ipv6-enp5s0.timer
[Timer]
OnBootSec=1min
OnUnitActiveSec=15min
Persistent=true

[Install]
WantedBy=timers.target
```

Cloud-init also calls the script once more right after `maas init`, because that step is exactly when MAAS might replay netplan.

Fifteen minutes is short enough to matter. A SLAAC address has to survive until the rack controller's next interface refresh before MAAS sees it. Prefix rotations happen on the scale of days, so a 15-minute window almost never overlaps one.

## What I took away

- **"Discovery is off" isn't the same as "MAAS won't learn new networks."** MAAS learns from its own controllers' interfaces regardless. If a controller's NIC can pick up addresses you don't control, MAAS will record them.
- **Residential IPv6 is a moving target.** Anything that keys state on observed prefixes will accumulate garbage on a rotating delegation. That includes IPAM, firewall objects and monitoring labels.
- **Boot-time config isn't runtime enforcement.** Anything another component can reset needs a reconciler, or it should live somewhere that component doesn't touch. It's the same lesson as Kubernetes controllers, at the scale of a single sysctl.
- **Unbounded growth eventually hits a hard limit.** Here it was AMP's 64 KB, but some limit always exists. A table that only grows deserves an alert long before something crashes.

All of this lives in the Terraform for my lab. The MAAS VM's cloud-init carries both fixes, so a rebuilt VM comes up already protected.
