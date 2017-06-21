# Getting started with ZFS on Ubuntu 16.04


This short tutorial will introduce you to ZFS, show how to install and 
configure it on your system and basic first steps.

This tutorial isn't here to sell you on the benefits of installing and
using ZFS - there are plenty of sources of information online that can
help you decide whether you want to use it or not! What we are aiming
to achieve here though is to install the necessary components,
configure them for use and to run through some essential commands.

By the end of this tutorial, you should be able to create different
kinds of storage pools, create a filesystem and delete both when
they are no longer needed.



# Installing ZFS

ZFS is fully supported on Ubuntu 16.04LTS(Xenial), but it is not
installed by default. You can install the latest package direct
from the archive with the following command:


```bash
sudo apt install -u zfsutils-linux
```

This will install the necessary components and the commands
(`zpool` and `zfs`) needed to make use of ZFS.


# Understanding Virtual Devices

Virtual Devices, or vdevs, are a key concept in ZFS, because
ZFS encompasses volume management as well as just a filesystem.
Essentially, a vdev is a kind of 'meta' device, that represents
one or more physical devices, accessed in a certain pattern. If
you are familiar with using RAID on Linux, this should not be
too unfamiliar - you may have a /dev/md0 which actually maps to
several physical disks.
There are several types of vdevs, including some special ones
related to caching and logging. The important ones for storing
data are:

 - **disk**:  dynamic-sized striped storage on given disks
(roughly equivalent to RAID0).
 - **mirror**: mirrored storage over given disks, equivalent
to RAID1.
 - **raidz1**: mirrored, striped storage with redundancy,
roughly equivlent to RAID5. Requires at least 3 disks.
 - **raidz2**: multiple redundancy array, roughly equivalent
to RAID6. Requires at least 4 disks.
 - **raidz3**: There is no particular equivalent for this mode,
which uses triple distributed parity to make recovering from
failure even faster.
 - **file** : This points to a file on a currently mounted
filesystem. Designed primarily for testing.

There are some additional, _special_ types worth mentioning:

 - **spare**: This is a disk that is allocated as a hot spare
in case of failure.
 - **cache**: This is space that is used as an L2ARC, we will
cover this later.
 - **log**: The ZFS intent log (or ZIL) is roughly equivalent
to a journal in ext4. Usually it is distributed amongst devices
in the pool, but for better performance it can be mounted
separately (e.g. on an SSD)

It is important to note that many of these can (and sometimes
should) be mixed when creating a pool, examples of which are
covered in the next section.


# Creating a pool

Storage in ZFS is orgnised into pools. Pools are created and
managed with the `zpool` command. This command needs root
privilege on Ubuntu, so it is usually run via `sudo`.

Let's assume you have four physical disks you wish to
allocate to a pool, sde,sdf,sdg,sdh. You can create simple striped
storage with the following command:

```bash
sudo zpool create newpool sde sdf sdg sdh
```

Note that in the case of a 'disk' VDEV, you can omit the
specification. It is also not necessary to specify the full
path to the devices you intend to use (though sometimes it is
useful to do so). You can specify partitions (e.g. sde1, sde2)
to be used, but this isn't generally desireable as it somewhat
negates the performance benefits on magnetic storage.

To check which pools are available, you can use the command:


```bash
sudo zpool list

```

Obviously, the output will depend on the drives you have
attached, but it should be similar to the following:

```
NAME     SIZE     ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
newpool  429.8G    70K   429.8G         -     0%     0%  1.00x  ONLINE  -
```

For greater detail, you can run the command:

```bash
sudo zpool status
```
which will output the current status of all pools, like so:

```
  pool: newpool
 state: ONLINE
  scan: none requested
config:

	NAME           STATE     READ WRITE CKSUM
	newpool        ONLINE       0     0     0
	  sde          ONLINE       0     0     0
	  sdf          ONLINE       0     0     0
	  sdg          ONLINE       0     0     0
	  sdh          ONLINE       0     0     0

errors: No known data errors
```

As you can see, this also groups the individul devices under the pools which
have been created. 

To create more example pools, you should now remove this one. This is done like so:

```bash
sudo zpool destroy newpool
```

!!! Note: If filesystems have been created in the pool, you won't be able to get rid of it so easily!

A mirrored pool is created in a similar way:


```bash
sudo zpool create testpool mirror sde sdf sdg sdh
```

In this case, running the status as before will reveal slightly different output:


```
  pool: testpool
 state: ONLINE
  scan: none requested
config:

	NAME             STATE     READ WRITE CKSUM
	testpool         ONLINE       0     0     0
	  mirror-0       ONLINE       0     0     0
            sde          ONLINE       0     0     0
	    sdf          ONLINE       0     0     0
	    sdg          ONLINE       0     0     0
	    sdh          ONLINE       0     0     0

errors: No known data errors
```

As you can see, in this case a pseudo-device, mirror-0, is listed as part of the pool.


# Complex pools

By using a combination of VDEVs, it should be possible to construct a pool
which fits well with your available hardware and your use case. This usually
involves using some of the special types of VDEV we mentioned earlier.

The cache VDEV in particular can make a huge difference, especially if you
have some fast storage. In normal operation, ZFS will use available RAM as an
ARC (Adaptive Replacement Cache), which keeps the most frequently read data
within easy reach. Only things which are not in the cache require the
filesystem to read from disk. But, if you have some fast storage available
(e.g. an ssd), it can operate as a second level cache (L2ARC) - not as fast
as memory, but still faster than reading from slower magnetic media.


When it comes to writing data, the ZIL can also benefit from a fast device.
As mentioned before, by default this log is spread out amongst the storage
devices, but if it is concentrated on a fast
