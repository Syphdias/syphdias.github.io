---
title: "Killing Ceph by Changing Networks"
date: 2025-11-05T20:11:02+01:00
lastmd: 2025-11-06T16:22:00+01:00
---
Since a customer requested to change networks of Ceph first in 2023 and then
two more times, I felt like I had to put it in writing some time.

This use-case _should_ be rare. But here the network _design_ changed multiple
times — maybe this is that _Agile_ I heard so much about. But in all
seriousness, network design should come first and not be subject to constant
change (unless you want to hire me).

When checking the Ceph documentation, I was pleasantly surprised! I made my fair
share of negative experiences with Ceph Docs (web and cli help) but this part
improved quite a bit for such a niche procedure.

In this I will go through the documented ways of how to change the network. I
will go through a more thorough example of the process including things not at
the core of the change and how to deal with them.

There are currently three documented ways to change Network/IPs.

## "The Right Way"

![Important: Existing monitors are not supposed to change their IP
addresses.](/assets/ceph-existing-mons.png)
_Screenshot from the official Ceph documentation_

While old documentation called it ["The Right Way"], it is now called the
["Preferred Method"].

I wouldn't even call it _changing a Ceph Mon's IP_ since it basically adds a new
Ceph Mon with a new IP, wait for the quorum, remove the old Ceph Mon, rinse and
repeat.

The documentation only suggest to do this in a manual fashion but in practice
[cephadm] and [rook] should make this trivially easy with changing the service
definition to relocate the Ceph Mons, as long as…

…the new IPs are within the current network or at least reachable.

## "The Messy Way"

While old documentation called it ["The Messy Way"], it is now called ["Advanced
Method"] — a bit boring, if you ask me since it can get messy, if you are
unprepared.

The high-level step by step is basically:

1. Stop the cluster
2. Export the monmap
3. Edit the monmap to remove/add Ceph Mons
4. Inject monmap into Ceph Mons
5. Start Mons, then rest of cluster

This leaves out a few things like changing the `public_network` in the
configuration database and how to deal with daemons and all the _messiness_ that
might come after.

## "The Third Way"

Okay, I lied. There is no third way but a more detailed version of ["The Messy
Way"](#the-messy-way) using cephadm.

The high-level steps are the same, but this time done within `cephadm shell`
including the resolution of some of the _messiness_ and it still assumes you
know how to get around your setup — if you don't, please abort and ask for help!

## My Take With My (Customer's) Idiosyncrasies

First off, **start with a healthy cluster**, if you can!

Secondly, **backup** monmap, configuration, and keyrings where you can reach it
(not /tmp of cephadm shell container or similar).

Make sure you are prepared and that you communicated the change — duh‽ I will
mention some things outside the cluster to look out for at the end.

I want to prevent data movement of any kind. It is probably not needed but I
like the precaution.

```sh
ceph osd set noout
ceph osd set norebalance
ceph osd set nobackfill
```

Stopping all Ceph services using Ansible and prevent them from starting when the
hosts will be rebooted for the network changes to be properly applied.

```sh
ansible ceph_hosts -m systemd -a 'name=ceph.target state=stopped enabled=false' -b
```

On one of the hosts I do more or less what ["The Third Way"](#the-third-way)
says. I change my monmap but with two important changes:

- Using a volume for the `cephadm shell` to easily extract monmap
- Only doing this once and then injecting the same monmap into all Ceph Mons
  while the documentation suggests repeating the edit process for all Ceph Mons

```console
root@ceph01:~# cephadm shell --name mon.ceph01 --mount /root:/roothome
Inferring fsid f52ba149-31ea-4da9-9b11-17e745e20d35
Inferring config /var/lib/ceph/f52ba149-31ea-4da9-9b11-17e745e20d35/mon.ceph01/config
Using recent ceph image quay.io/ceph/ceph@sha256:1b9158ce28975f95def6a0ad459fa19f1336506074267a4b47c1bd914a00fec0
root@ceph01:/# cd /roothome/
root@ceph01:/roothome# ceph-mon -i ceph01 --extract-monmap monmap-$(date +%F)
2025-11-05T08:56:17.050+0000 70bab6991834 -1 wrote monmap to monmap-2025-11-05
root@ceph01:/roothome# cp monmap-2025-11-05{,-new}
```

This is the time to go to another terminal, and **back up the old monmap** now!
(Maybe even do this a day earlier)

Now I removed the old IPs and add the new ones. I believe the up-to-date version
of the add command more complicated than it needed to be and opted for the old
syntax.

```console
root@ceph01:/roothome# monmaptool --print monmap-2025-11-05-new
monmaptool: monmap file monmap-2025-11-05-new
epoch 10
fsid f52ba149-31ea-4da9-9b11-17e745e20d35
last_changed 2024-05-12T12:42:35.759936+0000
created 2022-01-13T13:05:50.128859+0000
min_mon_release 18 (reef)
election_strategy: 1
0: [v2:10.10.10.1:3300/0,v1:10.10.10.1:6789/0] mon.ceph01
1: [v2:10.10.10.2:3300/0,v1:10.10.10.2:6789/0] mon.ceph02
2: [v2:10.10.10.3:3300/0,v1:10.10.10.3:6789/0] mon.ceph03
3: [v2:10.10.10.4:3300/0,v1:10.10.10.4:6789/0] mon.ceph04
4: [v2:10.10.10.5:3300/0,v1:10.10.10.5:6789/0] mon.ceph05
root@ceph01:/roothome# monmaptool --rm=ceph{01..05} monmap-2025-11-05-new
monmaptool: monmap file monmap-2025-11-05-new
monmaptool: removing ceph01
monmaptool: removing ceph02
monmaptool: removing ceph03
monmaptool: removing ceph04
monmaptool: removing ceph05
monmaptool: writing epoch 10 to monmap-2025-11-05-new (0 monitors)
root@ceph01:/roothome# monmaptool --add ceph01 192.168.160.1 --add ceph02 192.168.160.2 --add ceph03 192.168.160.3 --add ceph04 192.168.160.4 --add ceph05 192.168.160.5 monmap-2025-11-05-new
monmaptool: monmap file monmap-2025-11-05-new
monmaptool: writing epoch 10 to monmap-2025-11-05-new (6 monitors)
root@ceph01:/roothome# monmaptool --print monmap-2025-11-05-new
monmaptool: monmap file monmap-2025-11-05-new
epoch 10
fsid f52ba149-31ea-4da9-9b11-17e745e20d35
last_changed 2024-05-12T12:42:35.759936+0000
created 2022-01-13T13:05:50.128859+0000
min_mon_release 18 (reef)
election_strategy: 1
0: [v2:192.168.160.1:3300/0,v1:192.168.160.1:6789/0] mon.ceph01
1: [v2:192.168.160.2:3300/0,v1:192.168.160.2:6789/0] mon.ceph02
2: [v2:192.168.160.3:3300/0,v1:192.168.160.3:6789/0] mon.ceph03
3: [v2:192.168.160.4:3300/0,v1:192.168.160.4:6789/0] mon.ceph04
4: [v2:192.168.160.5:3300/0,v1:192.168.160.5:6789/0] mon.ceph05
```

Ensure this is absolutely correct. When I did this, I accidentally added another
Ceph Mon that did not exist. Since there were still enough Ceph Mons to have a
quorum, and this change needed to be done during a cluster downtime, I was not
worried about temporarily losing quorum. This would also only happen if two more
Ceph Mons failed or did not come up right away. I later [removed the extra Ceph
Mon][Ceph Mon Removal] after the existing Ceph Mons were up again.

I did not want to do these steps for every Ceph Mon host, so I copied the new
monmap to the relevant hosts.

```bash
for i in {02..05}; do
    ssh ceph01 sudo cat /root/monmap-2025-11-05-new \
    | ssh ceph$i sudo tee /root/monmap-2025-11-05-new > /dev/null
done
```

I ran the following commands to inject the monmap on each one of them, quick and
dirty as well.

```bash
for i in {01..05}; do
    ssh ceph$i "sudo cephadm shell --mount /root:/roothome --name mon.ceph$i \
        ceph-mon -i ceph$i --inject-monmap /roothome/monmap-2025-11-05-new"
done
```

I updated `/var/lib/ceph/{FSID}/mon.{MON}/config` on every host by replacing the
IPs in the config.

So far this was only preparations for the network change. In my case the network
configuration is managed by an Ansible role, and I already made the necessary
changes. This is the point of ~~no return~~ annoying rollback, if one would be
needed for any reason.

Note: For rolling back I would inject the old monmap for all Ceph Mons. After
changing the network configuration on the hosts and the switches, this might be
harder as I might not be able to easily connect to the hosts to deploy the old
network configuration.

```sh
ansible-playbook site.yaml -t ceph_networking -D
```

Since I am not in charge of the network hardware, this was the time to talk to
the networking people, so they would switch over to the new configuration.

```sh
ansible ceph_hosts -m reboot -b
```

After the reboots I checked the network using another Ansible script, that would
check for bond failures, negotiated speed, VLAN tags, and finally pinging from
every host to every other host on public and cluster network using jumbo frames.
Depending on the cluster size this takes a while but is worth the effort in my
experience to not run into weird issues due to missing MTU on some interfaces or
other configuration errors (hardware or software).

Note: To ping from everywhere to everywhere it builds a list of IPs filtered by
CIDRs and loops over this list on every host.

```sh
ansible-playbook check-network.yaml
```

Restart the cluster, if everything is okay.

```sh
ansible ceph_hosts -m systemd -a 'name=ceph.target state=started enabled=true' -b
```

Setting `public_network`. I am not sure why it is set on `mon` and `global` level,
but there is no harm updating both.

```bash
ceph config set mon public_network 192.168.160.128/24
ceph config set global public_network 192.168.160.128/24
```

Fixing Ceph Mgr config:

```bash
ceph config generate-minimal-conf >/var/lib/ceph/f52ba149-31ea-4da9-9b11-17e745e20d35/mgr.ceph01.yeinoh/config
systemctl reset-failed ceph-f52ba149-31ea-4da9-9b11-17e745e20d35@mgr.ceph01.yeinoh
systemctl start $_
```

Since the orchestrator does not know the new IPs yet, I needed to set them.

```sh
ceph orch host set-addr ceph01 192.168.160.1
# repeated for all hosts
```

It will take a few minutes for the orchestrator to gather all information from
every host again.

The documentation only mentions one last command to reconfigure the Ceph OSDs
before it leaves you with the rest to fix for yourself. For example, to get a
working standby Ceph Mgr and some other services I ran `ceph orch reconfig` for
all services on this cluster, including Ceph Mons to ensure there is no residue
config referencing the old network.

```sh
ceph orch reconfig osd
ceph orch reconfig mgr
ceph orch reconfig mds
ceph orch reconfig mon
ceph orch reconfig rgw
```

Note: I guess `ceph orch redeploy $service` would work as well but would take
longer.

Now we unset the flags to allow the cluster to function again normally.

```sh
ceph osd unset noout
ceph osd unset norebalance
ceph osd unset nobackfill
```

Now for the supporting services, that need to be checked for functionality and I
will try to include all relevant ones (including unused ones in this case):

- Ceph Mgr should provide Ceph Dashboard with proper certificates and Ceph
  Exporter
- Observability tooling should be functional: Prometheus, Node-Exporter, Loki,
  Promtail, Grafana, Alertmanager
- If you scrape from another external Prometheus you might need to change IPs to
  be scraped there
- If there are custom containers, e.g. S3 usage exporter, they should still be
  reachable
- Sync services functional: RGW Sync, RBD Mirror, and CephFS Mirror
- Change DNS RR(set)s for RGWs, etc.

Lastly the RADOS clients (RGW, RBD, CephFS) need to be reconfigured to be able
to communicate with the cluster again.

And if there are firewalls in place, their rules should be updated _beforehand_;
on hosts (iptables/nftables) and on centralized firewall appliances.

## Conclusion

While it is possible and fairly straight forward to move a cluster to another or
multiple other networks, I would advice avoiding the procedure. You might end up
with no cluster at all, if you are not able to fix issues along the way. Even if
you do not make mistakes during the process you might still encounter new
hurdles with later versions or something might fail during the process.

Understanding how to heal your cluster in an emergency is vital and you should
have the ability to do so, if you attempt this.

This procedure really should not be a regular occurrence, which is why I did not
automate it the first time I had to do it and not the second time either.

["The Right Way"]: https://docs.ceph.com/en/pacific/rados/operations/add-or-rm-mons/
["Preferred Method"]: https://docs.ceph.com/en/latest/rados/operations/add-or-rm-mons/#changing-a-monitor-s-ip-address-preferred-method
[cephadm]: https://docs.ceph.com/en/latest/cephadm/
[rook]: https://docs.ceph.com/en/latest/mgr/rook/
["The Messy Way"]: https://docs.ceph.com/en/pacific/rados/operations/add-or-rm-mons/#changing-a-monitor-s-ip-address-the-messy-way
["Advanced Method"]: https://docs.ceph.com/en/latest/rados/operations/add-or-rm-mons/#changing-a-monitor-s-ip-address-advanced-method
[Ceph Mon Removal]: https://docs.ceph.com/en/reef/rados/operations/add-or-rm-mons/#removing-a-monitor-manual
