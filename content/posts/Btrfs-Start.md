---
title: "My foray into Btrfs"
date: 2021-05-03
slug: "BtrfsFirstContact"
---
# My foray into Btrfs
## Why?
Fedora uses it as default for its Workstation install and I wanted to try it for a while as the snapshotting and bitrot-detection seemed interesting for me for a while.\
There seem to be some benefits to be gained from reinstalling my OS on Btrfs. This blog is meant to document my learning.

---
## What is Btrfs, really?
According to the [Btrfs-Wiki](https://btrfs.wiki.kernel.org/index.php/Main_Page):
> **Btrfs** is a modern copy on write (CoW) filesystem for Linux aimed at implementing advanced features while also focusing on fault tolerance, repair and easy administration.\
> Its main benefits are:
> - **Snapshots** which do not make the full copy of files (CoW)
> - **RAID** - support for software based RAID 0, RAID 1, RAID 10
> - **Self-healing** - checksums for data and metadata, automatic detection of silent data corruptions

That sounds all fine and dandy, but what exactly does all of that mean?\
*Copy on Write* means that between versions of a file only the deltas get written to the disk. So if there is a change in the file only the change gets written to the drive.
Therefore a lot of the information is saved in Metadata pointing at the right parts of data written on the file and how it should be assembled.
This makes it also easy to implement *lazy copies* where a copy is just a reference to the original file.
Copy on write is however not recommended for heavily updated-in-place files such as VM-images and database stores.
This is caused by the copy in place where you always keep the original in addition to the updated file.\
What is also attractive about Btrfs is the fact that the file system also handles volume management, compression, snapshotting without needing extras such as LVM.\
The snapshotting is allowing rollbacks to prior states of the file system, so if something gets messed up real bad you can just go back to a prior saved step. There are tools that make the interface with snapshotting easier, for example `snapper` (which also has integration in Fedoras `dnf` package manager via a plugin). More on that at a later point.\
Compression is a nice feature of Btrfs as it reduces file sizes, and who doesn't like to have more space on the harddrive.
This is also helpful in that it reduces wear on the drives, as there are fewer writes to disk.
It seems that the Fedora devs also think so as they enable transparent compression by default on install.\
The included checksumming/self-healing is useful as it keeps track of degrading data.
This is great for people like me who are always concerned about their data developing errors (I already lost some photos to bitrot).

These features in turn mean, that Btrfs is slightly less performant than other file systems like XFS or Ext4 as the overhead for the checksumming and compression does impact performance.

One thing that Btrfs does not manage itself is encryption. However this can easily be solved by using [`dm-crypt`](https://www.kernel.org/doc/html/latest/admin-guide/device-mapper/dm-crypt.html#), the transparend disk encription subsystem in the Linux kernel.

---
## Where to start?
A good start is the aforementioned [Btrfs-Wiki](https://btrfs.wiki.kernel.org/index.php/Main_Page) and for a more practical guide the [Arch-Wiki Page](https://wiki.archlinux.org/index.php/Btrfs).
For getting a good overview the [SysadminGuide](https://btrfs.wiki.kernel.org/index.php/SysadminGuide) is a great start.
And hopefully this article.

---
Based largely in on the Btrfs-Wikis' SysadminGuide I will now talk about important parts of the filesystem.
## Data usage/allocation
(Adapted from the [Btrfs-Wiki](https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Data_usage_and_allocation))\
The lowest level of the file system deals with a pool of raw storage made up of block devices containing the volume and its subvolumes.\
The size of this pool is reported by the `df`-command.\
The structure is that the volume needs storage to hold the data, which it allocates in chunks from the raw storage, typically in 1GiB increments. This allocation is what you then see in the output of `btrfs filesystem`.
A chunk is therefore a piece of storage Btrfs can put data on.
To cite:
> Terminology: space is _allocated_ to chunks, and space is *used* by blocks. A chunk with no used blocks in it is *unallocated*; a chunk with 1 or more used blocks in it is *allocated*. All chunks can be allocated even if not all space is used.
# Subvolumes
A subvolume is a independently mountable POSIX filetree, **not a block device**. 
The cool thing about Btrfs is that each top level subvolume or volume has a mountable root and that each volume can contain more than one filetree.\
When compared with LVM logical volumes Btrfs subvolumes are not similar, as the LVM logical volume is a block device in its own right containing its filesystem or container (dm-crypt, MD RAID...).\
Each subvolume root directory differs from a directory in that each subvolume has its distinct inode number space and each inode under a subvolume has a distinct device number.
## Subvolume Layout
There are two main/basic layouts to use: *flat* and *nested*.\

---
**Flat**: Here the subvolumes are children of the toplevel subvolume and are directly below the top level subvolume.\
This has several implications:
- Subvolume management is considered to be easier as the effective layout is more directly visible.
- All subvolumes need to be mounted manually (or via fstab) to the desired location
- Each subvolume can be mounted with some options being different, but all options must always be explicitely stated.
- Everthing in the volume that is **not** beneath a mounted subvolume is inaccessible. This can be beneficial for security/snapshots

---
**Nested**: Here we have subvolumes located anywhere in the file hierarchy. Typically in their desired locations, where they'd be typically mounted in the flat schema.\
The following implications arise:
- Management of snapshots is especially rolling them may be considered more difficult as the effective layout is not directly visible.
- Subvolumes don't need to be mounted manually, they appear automatically at the respective tree-base
- for each of the subvols the mount option of their mountpoint applies.
- Everything is visible
  
---
There is also the possibility to create a **mixed** layout such as a flat base layout with certain parts of the filesystem being put in nested subvolumes.
Here care must be taken with regard to snapshotting, there are no recursive snapshots, so nested subvolumes must be snapshotted separately.

## When to Make Subvolumes - Guidelines
**Nested Subvolumes** are **not** going to be part of the snapshots of their parent subvolume. This can be used to exclude certain parts of the filesystem from being snapshot (for example software builds).

To split areas that are complete/consistent in themselves.
- `/home`
- `/var/www`
- `/var/lib/postgresql`\
  these parts are more or less contained within them selves and "operate autonomously" from the rest of the subvolumes.

Splitting them when they "belong together" or depend in the state of each other (such as `/var/lib`, `/usr`, `lib`, `/bin`) is not recommended in conjunction with snapshots as this can lead to breakage.\
Splitting can be useful in cases of btrfs-send/-receive, as this feature works on subvolumes.\
Splitting might also be sensible in cases of special mount options.

# Snapshots
A could be described as saving the metadata of a subvolume at a specific point in time.
Therefore they are heavily reliant on the CoW capabilities of Btrfs.\
In the [Btrfs-Wiki](https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Snapshots) a Snapshot is described as:
> [...] a subvolume that shares its data (and metadata) with some other subvolume, using Btrfs's COW capabilities.

Once a snapshot is taken there is no difference between the original subvolume and the snapshot, after the snapshot the "original" subvolume is then written to and its layout changes.\
The snapshot can be rolled back to by unmounting and renaming the original subvolume and then renaming the snapshot to the original subvolumes name and mounting the snapshot.

Security issues arise when using snapshots as permissions that are made after a snapshot is made do not get "backported" to the snapshot.
Care has to be taken in such cases to not "leak" sensitive information etc. via snapshots.
This could arise in nested layouts, as snapshots may be visible to other users.

---
After all this theoretical background lets get a bit more practical.
# What did I do? - Exploration Phase
I installed Fedora 34 on a 500GB NVME-SSD on my desktop.
As I wanted to get going asap I chose to nuke and pave over my previous install and use the automatic partitioning.
What this then created was a layout that looks as follows:
```
NAME                             MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
nvme0n1                          259:0    0 476.9G  0 disk  
├─nvme0n1p1                      259:1    0   600M  0 part  /boot/efi
├─nvme0n1p2                      259:2    0     1G  0 part  /boot
└─nvme0n1p3                      259:3    0 475.4G  0 part  
  └─luks-many-numbers-here       253:0    0 475.3G  0 crypt /home
```
So we have a `/boot/efi`-partition that is a vfat formatted, a `/boot` partition that is ext4 formatted and the btrfs-partition containing everything else.
Now this is just a vague overview. When checking the [fedoraproject wiki](https://fedoraproject.org/wiki/Btrfs) and/or the fstab we find out that the default layout of the Btrfs-pool is two subvolumes, one for `/`(root) and one for `/home`.\
More information can be gathered via the `btrfs` command in the terminal.
## The `btrfs`-commands
In this part I lean heavily on the man-pages.
### `btrfs filesystem`
As an example here is the default `btrfs filesystem show output`:
```
Label: 'fedora_localhost-live'  uuid: uuid-goes-here-with-numbers
	Total devices 1 FS bytes used 25.34GiB
	devid    1 size 475.34GiB used 27.02GiB path /dev/mapper/luks-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```
We can see that the label of the filesystem is still the one of the live host, so how do we change it?\
`btrfs filesystem label` is the answer:
```
btrfs filesystem label [<device>|<mountpoint>] <newlabel>
```
Some other options that can be found in the `btrfs filesystem` command are `df -h` for the current allocation of block group types per mount point.
However a nice overview can be had by using `btrfs filesystem usage <path>`.
This command is supposed to replace the `df` command in the long run and when using it it is immediately appearent why, as it is much clearer.
```
btrfs filesystem usage /
Overall:
    Device size:            475.34GiB
    Device allocated:        27.02GiB
    Device unallocated:     448.32GiB
    Device missing:             0.00B
    Used:                    25.34GiB
    Free (estimated):       449.43GiB	(min: 449.43GiB)
    Free (statfs, df):      449.43GiB
    Data ratio:                  1.00
    Metadata ratio:              1.00
    Global reserve:          59.20MiB	(used: 0.00B)
    Multiple profiles:             no

Data,single: Size:26.01GiB, Used:24.89GiB (95.72%)
   /dev/mapper/luks-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx    26.01GiB

Metadata,single: Size:1.01GiB, Used:457.36MiB (44.32%)
   /dev/mapper/luks-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx     1.01GiB

System,single: Size:4.00MiB, Used:16.00KiB (0.39%)
   /dev/mapper/luks-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx     4.00MiB

Unallocated:
   /dev/mapper/luks-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   448.32GiB
```
The `Device size` gives the raw device capacity.\
The `Device allocated` is the sum of total space allcated for data/metadata/system profile, this also accounts for space reserved but not yet used for extents.\
The `Device unallocated` is the remaining unallocated space on the device, free for future allocations.\
The `Used` part is the sum of space that is currently used without any reserved space.
The `Free` parameters give the approximate size of the remaining space.\
The `Data ratio` represents the ratio of total space for data includin redundancy (single is 1.0, RAID1 is 2.0).\
The `Metadata ratio` is the equivalent for metadata.\
The `Global reserve` is the portion of metadata currently used for global block reserve, is only used for emergency purposes.

The lower four lines are visible when running the command as superuser. This shows the stats per block group types.

With `btrfs filesystem resize` mounted filesystems can be resized. Here the `<devid>` of the respective filesystem should be used.
This can be found in the output of the `btrfs filesystem show` command.
It defaults to 1 if left empty.\
The `<size>` parameter specifies the new size of the filesystem. With prefixes (+/-) the filesystem can be increased or decreased by the quantity given.
If the maximum available is passed the filesystem will occupy all the available space on the device. Growing is usually instantaneous, but shrinking can take a long time if there is a lot of data on the device, as the relocation of any data that is already written beyond the newly set limit needs to be relocated.\
**The resize command does not resize the underlying partition!** To do so you must use different tools such as `fdisk` or `parted`.

### `btrfs subvolume`
The `btrfs subvolume` command is the command that is used to manage your subvolumes and snapshots.

To list all subvolumes in a path simply enter `btrfs subvolume list <path>`:
```
ID 256 gen 19607 top level 5 path home
ID 257 gen 19602 top level 5 path root
ID 262 gen 18936 top level 257 path var/lib/machines
```
The `ID` is the subvolume-id, `gen` is a counter that is updated with every transaction. `top level` represents the id of the parent subvolume (5 is the topmost level) and `path` is the relative path of the subvolume to the top level subvolume.

Here we see that the default Fedora-Install sets up three subvolumes in a mixed layout, where root and home are in a *flat* layout and the `var/lib/machines` subvolume is nested within the `home`. 
This subvolume is created by SystemD for use as a backend for [`machinectl`](https://www.freedesktop.org/software/systemd/man/machinectl.html).

More commands that are of note in this command:
- `btrfs subvolume create` - is used to create a new subvolume.\
  Either it is given with a destination or the subvolume is created in the current directory.\
  The subvolume can also be addded to a quota-group via `-i <qgroupid>`
- `btrfs subvolume delete` is used to delete subvolumes and snapshots.\
  The deleted subvolume-directory is removed instantly, but the data-blocks are removed later in the background.
- `btrfs subvolume show` shows more detailed information about a subvolume such as its UUID, Creation time, Subvolume ID and Name.
- `btrfs subvolume snapshot` is used to create a snapshot of a subvolume.

### `btrfs quota`
The commands under `btrfs quota` are used to change the quotas of the filesystem.\
According to the manpage there might be some edge-cases where quotas may lead to instability.
In addition there are some performance implications to enabling quotas, so the activation is not recommended unless the user wants to use them.

These quotas are really useful in resource allocation in multi-user environment.
For a more detailed description the man-page of `btrfs-quota` is an excellent resource.
On a single-user machine such as my install qgroups can be used as a replacement for partitions, so you can determine how much space a given subvolume should contain.
This can be useful for restricting subvolumes to avoid them taking over the entire filesystem.
For example in case of logfiles that might get very big or if creating software that might get a bit overly excited and create loads of data.\
Setting and adjusting group quotas is done with the `btrfs qgroup` command.

For my install I did not (yet) find a use for implementing quotas.

### `btrfs device`
These commands are used to manage the devices in a Btrfs filesystem.\
Device management in Btrfs works on a mounted filesystem.
The devices can be added, removed or replaced (for replacement `btrfs replace` is used).\
These commands are especially useful if you want to add disks to your current filesystem.\
Another really useful set of stats can be called with the `btrfs device stats <path/to/device>`:
```
btrfs device stats <device>
[<device>].write_io_errs    0
[<device>].read_io_errs     0
[<device>].flush_io_errs    0
[<device>].corruption_errs  0
[<device>].generation_errs  0
```
`write_io_errs` records failed writes to the block device.
These errors are rooted in layers beneath the filesystem not being able to satisfy the write request.\
`read_io_errs` are the analogous errors for reads from the block device.\
`flush_io_errs` are the number of failed writes with the *FLUSH* flag set.
Flushing is a method of forcing a particular order between write requests, that is crucial for implementing crash consistency. In the case of btrfs, all metadata blocks **MUST** be permanently stored on the block device before the superblock is written.\
`corruption_errs` show up if a block checksum is mismatched or a corrupted metadata header is found.\
`generation_errs` occur when the block generateion does not match the expected value.

These errors are kept in a persistent record.
The current values are printed at mount time and updated during filesystem lifetime or from a scrub run.

Other commands found under `btrfs device` are to `add`, `remove` and `delete` devices from the filesystem and/or arrays as well as to check whether all devices in a multiple device filesystem are `ready` to be mounted.

### `btrfs scrub`
These commands are used to scrub a mounted filesystem, ergo read all data and metadata, and verify their checksums as well as automatically repair corrupted blocks if a correct copy is available.\
What scrub is **not** is a filesystem checker that verifys or repairs structural damage to the filesystem.

Results from a scrub look like the following:
```
btrfs scrub start / && btrfs scrub status /
UUID:             xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
Scrub started:    Fri Apr 30 12:18:53 2021
Status:           finished
Duration:         0:00:08
Total to scrub:   25.62GiB
Rate:             3.20GiB/s
Error summary:    no errors found
```

### More `btrfs`-commands
Other useful commands under `btrfs` are:
- `btrfs send` and `btrfs receive` to send data between subvolume snapshots.
- `btrfs balance` to spread block groups across devices
- `btrfs check`, `btrfs restore` and `btrfs rescue` if things go south.
---
# What did I do? Part II - Configuration changes
With all this background out of the way I have a basic understanding of the filesystem and the current layout of the disk I work on how I want it to be laid out for my use.

## Folders and `fstab`
For the most part the current layout works for me.
What I do want to do is to add a folder to my home in which to store my VMs.
As this is a folder for VMs I will disable the Copy on Write functionality as it is adviced not to use on large files with random writes.

To do this:
```
mkdir VMs
chattr +C VMs
lsattr
...
---------------C---- ./VMs
```
Here `chattr +C` defines that the copy on write functionality of btrfs is not used for the contents of the folder.
*Beware that this option only applies to files created in the directory after it is set, so best use is on empty directories.*

I chose this solution as the btrfs-man-page advices that seperate mount options are not supported for different subvolumes in the same filesystem.
Therefore creating a new subvolume and setting the `nodatacow` mount option is not an option.
In addition setting the `nodatacow` mount option would also prevent compression, as `nodatacow` implies `nodatasum`.

I then  realized that the fstab could be optimized slightly, as I can add the `ssd` option, as I know it's installed on an SSD.
Another useful flag for installations on SSDs is the `discard`-flag, especially with `discard=async` (if supported), as this allows the TRIM to operate when a block is freed.
Additionally I used the `noatime` (to prevent frequent disk writes) and `space_cache` (to improve performance when reading block group free space into memory) options.\
The fstab looks something like this:
```
UUID=... / btrfs subvol=root,ssd,noatime,space_cache,discard=async,compress=zstd:1,x-systemd.device-timeout=0 0 
```
After changing the fstab it is advised to let grub know about the changes.
For this we use:
```
grub2-mkconfig -o /boot/grub2/grub.cfg
```
*Note that since Fedora 34 the [grub configuration files are all in a unified location](https://fedoraproject.org/wiki/Changes/UnifyGrubConfig).*\
If you are on another distribution check where and how to change the grub on your system.

Reboot time.

After the reboot check whether all the changes worked (e.g. via `cat /etc/fstab`).

## Set up `btrfs scrub` to run regularly
This is one of the big advantages of btrfs: data integrity checking.
For this we use the `btrfs scrub`-command, but I don't want to always have to run it manually so I set up a SystemD-Service and -Timer to have it run regularly.\
I used this excellent [blog-post](https://www.jwillikers.com/btrfs-scrub) as a starting-point.

First off we want to create a systemd.service file.
Here I will use an [instantiated service](https://www.freedesktop.org/software/systemd/man/systemd.service.html#Service%20Templates), just to try.
This file will be created in the `/etc/systemd/system/` directory and will be named `btrfs-scrub@.service`.
```
[Unit]
Description=btrfs-scrub on %f 
ConditionPathIsMountPoint=%f
RequiresMountsFor=%f

[Service]
Nice=19
IOScedulingClass=idle
KillSignal=SIGINT
ExecStart=/usr/sbin/btrfs scrub start -B %f
```

Then we need a timer unit that runs the service regularly.
I decided to let it run once a week.
```
[Unit]
Description=btrfs-scrub on %f once a week

[Timer]
OnCalendar=weekly
Persistent=true

[Install]
WantedBy=timers.target
```

When these files are created it is best to then run a `systemd-analyze verify <files>` .
Then we `enable` the service via `systemctl start btrfs-scrub@-.service` and start the timer via `systemctl enable btrfs-scrub@-.timer`.
The `-` in the files represents the `systemd-escape`d `/`.

This concludes the basic btrfs-configuration that I undertook.