---
title: "Killing your Ceph with Backfilling"
date: 2023-01-11T22:42:15+01:00
---
I recently was consulted on a Ceph Cluster running into nearfull and backfillfull
for the first time. One Ceph OSD was utilized over 85% and another over 90%. The
operators were unaware of the meaning and what to do about it, so took a look.

Looking at `ceph status` and `ceph df`, I noticed something. Try to spot it
yourself – I made it easier by removing some stuff around it:

```
$ ceph status
[...]
    health: HEALTH_WARN
            1 pools have many more objects per pg than average
            1 backfillfull osd(s)
            1 nearfull osd(s)
            Low space hindering backfill (add storage if this doesn't resolve itself): 1 pg backfill_toofull
            1 pgs not deep-scrubbed in time
            1 pgs not scrubbed in time
            20 pool(s) backfillfull

  services:
[...]
    osd: 96 osds: 96 up (since 4w), 96 in (since 12M); 1 remapped pgs
[...]
  data:
    volumes: 4/4 healthy
    pools:   20 pools, 4769 pgs
    objects: 31.90M objects, 117 TiB
    usage:   351 TiB used, 522 TiB / 873 TiB avail
    pgs:     299136/95689743 objects misplaced (0.313%)
             4763 active+clean
             5    active+clean+scrubbing+deep
             1    active+remapped+backfill_toofull
[...]
# ceph df
--- RAW STORAGE ---
CLASS     SIZE    AVAIL     USED  RAW USED  %RAW USED
hdd    873 TiB  522 TiB  351 TiB   351 TiB      40.19
TOTAL  873 TiB  522 TiB  351 TiB   351 TiB      40.19

--- POOLS ---
POOL                           ID   PGS   STORED  OBJECTS     USED  %USED  MAX AVAIL
device_health_metrics           1  4096  907 MiB      108  2.7 GiB      0     14 TiB
rbd                             4   145   97 TiB   25.39M  290 TiB  87.43     14 TiB
[...]
```

The raw usage was only at 40%. Why would one disk contain so much data? The
balancer was in upmap mode and active. But even with no balancer, this kind of
miss-balancing would be extreme and very unlikely.

You may have already spotted something odd in the Ceph Pool configuration. While
`device_health_metrics` contained less than 1GiB, it had 4096 PGs. At the same
time `rbd` contained 97TiB in just 145 PGs.

145 is not just no power of two (which would usually produce a Ceph Warning),
but also way to low for the about of data and Ceph OSD count.


## What Does This Mean for Storage Distribution?

Estimating the size of one PG for pool `rbd` yields about 685GiB (97TiB/145).
How many (average sized) PGs will lead to utilization of one disk over 85%?

About 11.4 (85% \* 9TiB / 685GiB)

Unfortunately, not every PG is the same size. Looking at the sizes, multiple PG
exceed 800GiB. Furthermore not every Ceph OSD receives the same amount of PGs.
And as we will soon see the number of PGs was trying to get lower.


## Cause and Distributing Data

But what actually caused the bogus PG numbers? The answer is: The PG Autocaler.
For some reason only `device_health_metrics` set a `target_size_ratio` to 0.1.
This lead to the effective ratio to be 1 for this pool. Apparently the autoscaling
assumed this would mean all data would be stored in this pool. This also
explained why the number of PGs was not a power of two. The autoscaler set
`target_pg_num` to 32 to reduce the pool `rbd` even more. This was why there was
not Ceph Warning. This also means that if there were no disks were running full
right now, it certainly would have happened in the following days.

Before removing the ratio, I wanted to know what would happen. I disabled
autoscaling (`ceph osd pool set noautoscale`) and removed the target ratio:
```
# ceph osd pool set device_health_metrics target_size_ratio 0
# ceph osd pool autoscale-status
POOL                             SIZE  TARGET SIZE  RATE  RAW CAPACITY   RATIO  TARGET RATIO  EFFECTIVE RATIO  BIAS  PG_NUM  NEW PG_NUM  AUTOSCALE  BULK
device_health_metrics          906.5M                3.0        873.1T  0.0000                                  1.0    4096           1  on         False
rbd                            99048G                3.0        873.1T  0.3323                                  1.0      32        1024  on         False
[...]
```

This was a lot better and we decided to re-enable autoscaling (`ceph osd pool
unset noautoscale`) right away.

After a few minutes the backfillfull was gone. Soon to be followed by the
neafull. After a few days of rebalancing both pools had the proper PG count.

I am not sure why the ratio was set and why it was interpreted as it was. The
[docs](https://docs.ceph.com/en/latest/rados/operations/placement-groups/#viewing-pg-scaling-recommendations)
suggest this would not be a problem and I could not reproduce the behavior in a
more recent version of Ceph. So this was potentially fixed already.
