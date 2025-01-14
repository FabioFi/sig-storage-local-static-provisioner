# Operations

This document provides guides on how to manage local volumes in your Kubernetes
cluster. Before managing local volumes on your cluster nodes, here are some
configuration requirements you must know:

* The local-volume plugin expects paths to be stable, including across
  reboots and when disks are added or removed.
* The static provisioner only discovers either mount points (for Filesystem mode volumes)
  or symbolic links (for Block mode volumes). For directory-based local volumes, they
  must be bind-mounted into the discovery directories.

Glossary:

- _discovery directory_: Directory on the host from which provisioner will
  discover both filesystem and block local PVs.
- _provisioner_: In our documents, provisioner indicates local-volume-provisioner program.
- *local PV*: Kubernetes local persistent volume.
- *filesystem local PV*: Local PV with Filesystem mode
- *block local PV*: Local PV with Block mode

## Table of Contents

- [Create a directory for provisioner discovering](#create-a-directory-for-provisioner-discovering)
- [Prepare and set up local volumes in discovery directory](#prepare-and-set-up-local-volumes-in-discovery-directory)
  * [Use a whole disk as a filesystem PV](#use-a-whole-disk-as-a-filesystem-pv)
  * [Sharing a disk filesystem by multiple filesystem PVs](#sharing-a-disk-filesystem-by-multiple-filesystem-pvs)
  * [Link devices into directory to be discovered as block PVs](#link-devices-into-directory-to-be-discovered-as-block-pvs)
  * [Separate disk into multiple partitions](#separate-disk-into-multiple-partitions)
- [Deleting/removing the underlying volume](#deletingremoving-the-underlying-volume)

## Create a directory for provisioner discovering

```
$ sudo mkdir -p /media/ssdwd
```

NOTE: 

- We use `/media/ssdwd` as an example, but you can use any directory 
- This directory is configured in `hostDir` field in provisioner configuration
  and can only be configured for one storage class
- If you want to configure more than one local storage class, create one
  directory for each storage class

## Prepare and set up local volumes in discovery directory

After you prepared discovery directory, you can set up local
volumes to be discovered by provisioner.

### Use a whole disk as a filesystem PV

If you attached a disk onto your machine (e.g `/dev/sda`). You
can format and mount it into discovery directory with the following commands:

1) Format and mount

```
$ sudo mkfs.ext4 /dev/sda
$ DISK_UUID=$(blkid -s UUID -o value /dev/sda) 
$ sudo mkdir /media/ssdwd/$DISK_UUID
$ sudo mount -t ext4 /dev/sda /media/ssdwd/$DISK_UUID
```

2) Persistent mount entry into /etc/fstab

```
$ echo UUID=`sudo blkid -s UUID -o value /dev/sda` /media/ssdwd/$DISK_UUID ext4 defaults 0 2 | sudo tee -a /etc/fstab
```

NOTE:

- We use `/dev/sda` as a disk example, change it to real path of your
  device.
- You can also adjust filesystem and mount options as you wish
- It's best practice to use UUID both in fstab entries and the directory name
  of mount point, see our [best practices](best-practices.md).
- Using a whole disk is best practice if you need IO isolation
- On some cloud platforms, they may provide mechanism to format and mount local
  disks automatically which is recommended. Please refer to cloud platform
  documentation.

### Sharing a disk filesystem by multiple filesystem PVs

Instead of mount root of disk filesystem into discovery directory, you can
create multiple directory in disk, and bind mount them into discovery
directory. By doing this, a disk can be shared by multiple local filesystem
PVs. Here is an example:

1) Format and mount

```
$ sudo mkfs.ext4 /dev/sda
$ DISK_UUID=$(blkid -s UUID -o value /dev/sda) #not working
$ blkid -s UUID -o value /dev/sda
$ DISK_UUID = copy and paste last command result
$ sudo mkdir /media/$DISK_UUID
$ sudo mount -t ext4 /dev/sda /media/$DISK_UUID
```

NOTE: we should not mount disk into discovery directory.

2) Persistent mount entry into /etc/fstab

```
$ echo UUID=`sudo blkid -s UUID -o value /dev/sda` /media/$DISK_UUID ext4 defaults 0 2 | sudo tee -a /etc/fstab
```

3) Create multiple directories and bind mount them into discovery directory

```
#declare DISK_UUID
DISK_UUID = paste the value
for i in $(seq 1 10); do
  sudo mkdir -p /media/${DISK_UUID}/vol${i} /media/disks/${DISK_UUID}_vol${i}
  sudo mount --bind /media/${DISK_UUID}/vol${i} /media/disks/${DISK_UUID}_vol${i}
done
```

4) Persistent bind mount entries into /etc/fstab

```
#declare DISK_UUID
DISK_UUID = paste the value
for i in $(seq 1 10); do
  echo /media/${DISK_UUID}/vol${i} /media/disks/${DISK_UUID}_vol${i} none bind 0 0 | sudo tee -a /etc/fstab
done
```

NOTE:

- Local PVs sharing one disk filesystem will have same capacity and will have
  no capacity isolation. If you want to separate a disk into multiple PVs with
  capacity isolation. You can [separate disk into multiple
  partitions](#separate-disk-into-multiple-partitions) first.

### Link devices into directory to be discovered as block PVs

If you want to use block devices directly, you can simply link them into
discovery directory.

For safety, you must use the unique path of device.

Find unique path of device:

```
$ ls -l /dev/disk/by-id/
lrwxrwxrwx 1 root root  9 Apr 18 14:26 lvm-pv-uuid-kdWgMJ-OOfq-ox5N-ie4E-NU2h-8zPJ-edX1Og -> ../../sde
lrwxrwxrwx 1 root root  9 Apr 18 14:26 lvm-pv-uuid-VqD1G2-upe2-Xnek-PdXD-mkOT-LhSv-rUV2is -> ../../sdc
lrwxrwxrwx 1 root root  9 Apr 18 14:26 lvm-pv-uuid-yyTnct-TpUS-U93g-JoFs-6seh-Yy29-Dn6Irf -> ../../sdb
```

For example, if you want to use `/dev/sda`, you must link
`/dev/disk/by-id/lvm-pv-uuid-yyTnct-TpUS-U93g-JoFs-6seh-Yy29-Dn6Irf` not 
`/dev/sda`.

Link it into discovery directory:

```
$ sudo ln -s /dev/disk/by-id/lvm-pv-uuid-yyTnct-TpUS-U93g-JoFs-6seh-Yy29-Dn6Irf /media/disks
```

### Separate disk into multiple partitions

You can use [parted](https://www.gnu.org/s/parted/manual/parted.html) or other
tools to separate your disk into multiple partitions. This helps to isolate
capacity. Here is an example:

```
sudo parted --script /dev/sda \
    mklabel gpt \
    mkpart primary 1MiB 100000MiB \
    mkpart primary 100000MiB 200000MiB \
    mkpart primary 200000MiB 300000MiB \
    mkpart primary 300000MiB 400000MiB \
    mkpart primary 400000MiB 500000MiB \
    mkpart primary 500000MiB 600000MiB \
    mkpart primary 600000MiB 700000MiB \
    mkpart primary 700000MiB 800000MiB \
    mkpart primary 800000MiB 900000MiB


sudo parted /dev/sda print
```

NOTE:

- Partition disk is dangerous, please check your command carefully, use at your own risk
- Adjust arguments according to your needs, refer to [parted manual](https://www.gnu.org/s/parted/manual/parted.html)

After disks are partitioned, for each partition, you can follow above operation
guide to format and discover as filesystem volume or use as block device.

## Deleting/removing the underlying volume

When you want to decommission the local volume, here is a possible workflow.

1. Stop the pods that are using the volume
2. Remove the local volume from the node (ie unmounting, pulling out the disk, remove mount entries from /etc/fstab, etc)
3. Delete the PVC
4. The provisioner will try to cleanup the volume, but will fail since the volume no longer exists
5. Manually delete the PV object
