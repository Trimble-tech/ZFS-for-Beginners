# ZFS for Beginners

## What is ZFS?
ZFS, according to [Wikipedia](https://en.wikipedia.org/wiki/ZFS), *"is a filesystem with volume management capabilities"*. While other systems like ext4, btrfs, or xfs are focused on one disk, ZFS is intended to manage multiple disks. It has functionality similar to RAID but runs entirely in software and includes features to prevent corruption, backup or restore files, and compress files natively on the filesystem. Additionally, special folders called datasets can be created to tune the filesystem for different workloads or purposes. ZFS can also make use of RAM or special disk layouts for improved performance. This document intends to be an introduction to ZFS but give a wide picture to what it is capable of.

### License
- CDDL (Common Development and Distribution License) 
- Since [CDDL](https://openzfs.github.io/openzfs-docs/License.html) and [GNU GPL](https://www.gnu.org/licenses/gpl-3.0.en.html) are exclusive, software licensed only as one cannot be directly shipped with the other. This is why ZFS is not a part of the Linux kernel or installed by default.

## Disk Layouts
The way in which the disks are configured will depend on your budget, space, and the importance of your data. If you want speed or maximum space, you can use no redundancy which creates essentially a RAID 0. However, this would mean loosing all data if one drive fails. If you value the data, you can create a pool with varying levels of redundancy. You may also decide to have multiple pools if you provide enough disks.

## Creating a pool
When creating a pool, it is preferred to use drive identifiers for better stability over time, but the /dev/sdx... lettering works too.
   
    zpool create poolname \
        mirror \
            /dev/disk/by-id/id-here \
            /dev/disk/by-id/idhere2

### Stripe
    zpool create poolname /dev/sda

### Mirror: Single parity
    zpool create poolname mirror /dev/sda /dev/sdb

### RAIDz2: Double-parity
    zpool create poolname raidz2 /dev/sda /dev/sdb

## Destroying a pool
    zpool destroy poolname

## Monitoring a pool
- Overall Status
    
        zpool status

- Disk I/O every 5 seconds
    
        zpool iostat -v 5

- List dataset usage
    
        zfs list

- Get all of the pool information
    
        zpool get all poolname

- Find the current compression ratio
    
        zfs get compressratio poolname

## Setting properties for the pool
- Enable LZ4 compression (recommended for almost everything)

        zfs set compression=lz4 poolname

- Set a reasonable record size for general use
        
        zfs set recordsize=128k poolname

- Enable extended attributes

        zfs set xattr=sa poolname

- Disable access time updates
        
        zfs set atime=off poolname

## Datasets
- these reside inside a pool and can have their own settings
    - compression
    - quotas
    - snapshots

### Create a dataset
    zfs create poolname/datasetname

#### Set properties for different datasets
    zfs set recordsize=512mb poolname/media
    zfs set recordsize=128k poolname/documents
    zfs set recordsize=64k poolname/vms

## Snapshots
- a backup system to restore changes to files
- create versions which can be named and/or dated
- snapshots take relatively little space but can revert unwanted changes or corruption on a filesystem level

### Take a snapshot
    zfs snapshot poolname/documents@2026-07-14

### List snapshots
    zfs list -t snapshot

### See how much space a snapshot is using
    zfs list -t snapshot -o name,used,refer

### Roll back to a snapshot (destroys all changes since)
    zfs rollback poolname/documents@2026-07-14

### Clone a snapshot into a new dataset (for testing)
    zfs clone poolname/documents@2026-07-14 poolname/documents-test@2026-07-14

## Delete a snapshot
    zfs destroy poolname/documents@2026-07-14

- Snapshots can be configured to occur on a schedule with a third party program such as [Sanoid](https://github.com/jimsalterjrs/sanoid)
- Snapshots technically can be scheduled with cron or systemd, but naming and complexity makes it best for scripts or programs to manage

## Scrubs
- read all blocks in pool and verify checksums
- only stores changes, and will repair files if corrupted

    zpool scrub poolname

- Scrubs can be scheduled through a systemd or cron timer

## Expansion
It is best practice to either switch to bigger disks or to add vdevs.
- A VDev is a virtual device with hardware attached, an example of adding a mirror would be:
    
        zpool add poolname mirror \
            /dev/disk/by-id/id-here3 \
            /dev/disk/by-id/id-here4
    
- Replace a failed disk:
    
        zpool replace poolname \
            /dev/disk/by-id/id-here_old \
            /dev/disk/by-id/id-here_new
- The replace action can be considered a "resilver" process
    - copies all data over in a single location after checking, instead of just verifying like a scrub
    - may take a long time on large drives, it is important to have higher redundancy levels to prevent failures while replacing drives

## ZFS Send/Receive
- Backups of a pool can be pushed to a remote location using ZFS natively
- Performance is faster than similar tools like Rsync
- For encrypted pools, encryption keys never go over the network

        sudo zfs send -i poolname/data@old poolname/data@new | ssh user@backup-server zfs receive poolname/backup