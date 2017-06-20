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

Virtual Devices, or VDEVs, are a key concept in ZFS, because ZFS encompasses volume management as well as just a filesystem. Essentially, a VDEV is a kind of 'meta' device, that represents one or more physical devices, accessed in a certain pattern. If you are familiar with using RAID on Linux, this should not be too unfamiliar - you may have a /dev/md0 which actually maps to several physical disks.
There are several types of VDEVS, including some special ones related to caching and logging. The important ones for storing data are:

 - **disk**:  dynamic-sized striped storage on given disks (roughly equivalent to RAID0).
 - **mirror**: mirrored storage over given disks, equivalent to RAID1.
 - **raidz1**: mirrored, striped storage with redundancy, roughly equivlent to RAID5. Requires at least 3 disks.
 - **raidz2**: multiple redundancy array, roughly equivalent to RAID6. Requires at least 4 disks.
 - **raidz3**: There is no particular equivalent for this mode, which uses triple distributed parity to make recovering from failure even faster.
 - **file** : This points to a file on a currently mounted filesystem. Designed primarily for testing.

There are some additional, _special_ types worth mentioning:

 - **spare**: This is a disk that is allocated as a hot spare in case of failure.
 - **cache**: This is space that is used as an L2ARC, we will cover this later.
 - **log**: The ZFS intent log (or ZIL) is roughly equivalent to a journal in ext4. Usually it is distributed amongst devices in the pool, but for better performance it can be mounted separately (e.g. on an SSD)

It is important to note that many of these can (and sometimes should) be mixed when creating a pool, examples of which are covered in the next section.


# Creating a pool

Storage in ZFS is orgnised into pools.

Pools are created and managed with the `zpool` command. This command needs root privilege on Ubuntu, so it is usually run via `sudo`.

Let's assume you have three physical disks you wish to allocate to a pool, sdf,sdg,sdh. You can create simple striped storage with the following command:

```bash
sudo zpool create newpool sdf sdg sdh
```

Note that in the case of a 'disk' VDEV, you can omit the specification. It is also not necessary to specify the full path to the devices you intend to use (though sometimes it is useful to do so).

