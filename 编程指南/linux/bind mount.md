# Linux 中的绑定装载(Bind Mount)

A _bind mount_ is an alternate view of a directory tree. Classically, mounting creates a view of a storage device as a directory tree. A bind mount instead takes an existing directory tree and replicates it under a different point. The directories and files in the bind mount are the same as the original. Any modification on one side is immediately reflected on the other side, since the two views show the same data.

For example, after issuing the Linux command-

```
mount --bind /some/where /else/where
```

the directories `/some/where` and `/else/where` have the same content, which is the content of `/some/where`. (If `/else/where` was not empty, its previous content is now hidden.)

Unlike a hard link or symbolic link, a bind mount doesn't affect what is stored on the filesystem. It's a property of the live system.

## How do I create a bind mount?

### bindfs

The [`bindfs`](http://bindfs.org/) filesystem is a [FUSE](http://en.wikipedia.org/wiki/Filesystem_in_Userspace) filesystem which creates a view of a directory tree. For example, the command

```
bindfs /some/where /else/where
```

makes `/else/where` a mount point under which the contents of `/some/where` are visible.

Since bindfs is a separate filesystem, the files `/some/where/foo` and `/else/where/foo` appear as different files to applications (the bindfs filesystem has its own `st_dev` value). Any change on one side is “magically” reflected on the other side, but the fact that the files are the same is only apparent when one knows how bindfs operates.

Bindfs has no knowledge of mount points, so if there is a mount point under `/some/where`, it appears as just another directory under `/else/where`. Mounting or unmounting a filesystem underneath `/some/where` appears under `/else/where` as a change of the corresponding directory.

Bindfs can alter some of the file metadata: it can show fake permissions and ownership for files. See the [manual](http://bindfs.org/docs/bindfs.1.html) for details, and see below for examples.

A bindfs filesystem can be mounted as a non-root user, you only need the privilege to mount FUSE filesystems. Depending on your distribution, this may require being in the `fuse` group or be allowed to all users. To unmount a FUSE filesystem, use `fusermount -u` instead of `umount`, e.g.

```
fusermount -u /else/where
```

### nullfs

FreeBSD provides the [`nullfs`](https://www.freebsd.org/cgi/man.cgi?query=nullfs&sektion=5) filesystem which creates an alternate view of a filesystem. The following two commands are equivalent:

```
mount -t nullfs /some/where /else/where
mount_nullfs /some/where /else/where
```

After issuing either command, `/else/where` becomes a mount point at which the contents of `/some/where` are visible.

Since nullfs is a separate filesystem, the files `/some/where/foo` and `/else/where/foo` appear as different files to applications (the nullfs filesystem has its own `st_dev` value). Any change on one side is “magically” reflected on the other side, but the fact that the files are the same is only apparent when one knows how nullfs operates.

Unlike the FUSE bindfs, which acts at the level of the directory tree, FreeBSD's nullfs acts deeper in the kernel, so mount points under `/else/where` are not visible: only the tree that is part of the same mount point as `/some/where` is reflected under `/else/where`.

The nullfs filesystem may be usable under other BSD variants (OS X, OpenBSD, NetBSD) but it is not compiled as part of the default system.

### Linux bind mount

Under Linux, bind mounts are available as a kernel feature. You can create one with the [`mount`](http://man7.org/linux/man-pages/man8/mount.8.html)command, by passing either the `--bind` command line option or the `bind` mount option. The following two commands are equivalent:

```
mount --bind /some/where /else/where
mount -o bind /some/where /else/where
```

Here, the “device” `/some/where` is not a disk partition like in the case of an on-disk filesystem, but an existing directory. The mount point `/else/where` must be an existing directory as usual. Note that no filesystem type is specified either way: making a bind mount doesn't involve a filesystem driver, it copies the kernel data structures from the original mount.

`mount --bind` also support mounting a non-directory onto a non-directory: `/some/where` can be a regular file (in which case `/else/where` needs to be a regular file too).

A Linux bind mount is mostly indistinguishable from the original. The command `df -T /else/where` shows the same device and the same filesystem type as `df -T /some/where`. The files `/some/where/foo` and `/else/where/foo` are indistinguishable, as if they were hard links. It is possible to unmount `/some/where`, in which case `/else/where` remains mounted.

With older kernels (I don't know exactly when, I think until some 3.x), bind mounts were truly indistinguishable from the original. Recent kernels do track bind mounts and expose the information through <code/proc/_PID_/mountinfo, which allows [`findmnt` to indicate bind mount as such](https://unix.stackexchange.com/questions/295525/how-is-findmnt-able-to-list-bind-mounts).

You can put bind mount entries in `/etc/fstab`. Just include `bind` (or `rbind` etc.) in the options, together with any other options you want. The “device” is the existing tree. The filesystem column can contain `none` or `bind` (it's ignored, but using a filesystem name would be confusing). For example:

```
/some/where /readonly/view none bind,ro
```

If there are mount points under `/some/where`, their contents are not visible under `/else/where`. Instead of `bind`, you can use `rbind`, also replicate mount points underneath `/some/where`. For example, if `/some/where/mnt` is a mount point then

```
mount --rbind /some/where /else/where
```

is equivalent to

```
mount --bind /some/where /else/where
mount --bind /some/where/mnt /else/where/mnt
```

In addition, Linux allows mounts to be declared as _shared_, _slave_, _private_ or _unbindable_. This affects whether that mount operation is reflected under a bind mount that replicates the mount point. For more details, see [the kernel documentation](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt).

Linux also provides a way to move mounts: where `--bind` copies, `--move` moves a mount point.

It is possible to have different mount options in two bind-mounted directories. There is a quirk, however: making the bind mount and setting the mount options cannot be done atomically, they have to be two successive operations. (Older kernels did not allow this.) For example, the following commands create a read-only view, but there is a small window of time during which `/else/where`is read-write:

```
mount --bind /some/where /else/where
mount -o remount,ro,bind /else/where
```

### I can't get bind mounts to work!

If your system doesn't support FUSE, a classical trick to achieve the same effect is to run an NFS server, make it export the files you want to expose (allowing access to `localhost`) and mount them on the same machine. This has a significant overhead in terms of memory and performance, so bind mounts have a definite advantage where available (which is on most Unix variants thanks to FUSE).

## Use cases

### Read-only view

It can be useful to create a read-only view of a filesystem, either for security reasons or just as a layer of safety to ensure that you won't accidentally modify it.

With bindfs:

```
bindfs -r /some/where /mnt/readonly
```

With Linux, the simple way:

```
mount --bind /some/where /mnt/readonly
mount -o remount,ro,bind /mnt/readonly
```

This leaves a short interval of time during which `/mnt/readonly` is read-write. If this is a security concern, first create the bind mount in a directory that only root can access, make it read-only, then move it to a public mount point. In the snippet below, note that it's important that `/root/private`(the directory above the mount point) is private; the original permissions on `/root/private/mnt`are irrelevant since they are hidden behind the mount point.

```
mkdir -p /root/private/mnt
chmod 700 /root/private
mount --bind /some/where /root/private/mnt
mount -o remount,ro,bind /root/private/mnt
mount --move /root/private/mnt /mnt/readonly
```

### Remapping users and groups

Filesystems record users and groups by their numerical ID. Sometimes you end up with multiple systems which assign different user IDs to the same person. This is not a problem with network access, but it makes user IDs meaningless when you carry data from one system to another on a disk. Suppose that you have a disk created with a multi-user filesystem (e.g. ext4, btrfs, zfs, UFS, …) on a system where Alice has user ID 1000 and Bob has user ID 1001, and you want to make that disk accessible on a system where Alice has user ID 1001 and Bob has user ID 1000. If you mount the disk directly, Alice's files will appear as owned by Bob (because the user ID is 1001) and Bob's files will appear as owned by Alice (because the user ID is 1000).

You can use bindfs to remap user IDs. First mount the disk partition in a private directory, where only root can access it. Then create a bindfs view in a public area, with user ID and group ID remapping that swaps Alice's and Bob's user IDs and group IDs.

```
mkdir -p /root/private/alice_disk /media/alice_disk
chmod 700 /root/private
mount /dev/sdb1 /root/private/alice_disk
bindfs --map=1000/1001:1001/1000:@1000/1001:@1001/1000 /root/private/alice_disk /media/alice_disk
```

See [How does one permissibly access files on non-booted system's user's home folder?](https://unix.stackexchange.com/questions/190866/how-does-one-permissibly-access-files-on-non-booted-systems-users-home-folder) and [mount --bind other user as myself](https://unix.stackexchange.com/questions/115377/mount-bind-other-user-as-myself) another examples.

### Mounting in a jail or container

A [chroot jail](http://en.wikipedia.org/wiki/Chroot) or [container](https://en.wikipedia.org/wiki/OS-level_virtualisation) runs a process in a subtree of the system's directory tree. This can be useful to run a program with restricted access, e.g. run a network server with access to only its own files and the files that it serves, but not to other data stored on the same computer). A limitation of chroot is that the program is confined to one subtree: it can't access independent subtrees. Bind mounts allow grafting other subtrees onto that main tree. This makes them fundamental to most practical usage of containers under Linux.

For example, suppose that a machine runs a service `/usr/sbin/somethingd` which should only have access to data under `/var/lib/something`. The smallest directory tree that contains both of these files is the root. How can the service be confined? One possibility is to make hard links to all the files that the service needs (at least `/usr/sbin/somethingd` and several shared libraries) under `/var/lib/something`. But this is cumbersome (the hard links need to be updated whenever a file is upgraded), and doesn't work if `/var/lib/something` and `/usr` are on different filesystems. A better solution is to create an ad hoc root and populate it with using mounts:

```
mkdir /run/something
cd /run/something
mkdir -p etc/something lib usr/lib usr/sbin var/lib/something
mount --bind /etc/something etc/something
mount --bind /lib lib
mount --bind /usr/lib usr/lib
mount --bind /usr/sbin usr/sbin
mount --bind /var/lib/something var/lib/something
mount -o remount,ro,bind etc/something
mount -o remount,ro,bind lib
mount -o remount,ro,bind usr/lib
mount -o remount,ro,bind usr/sbin
chroot . /usr/sbin/somethingd &
```

Linux's [mount namespaces](http://lwn.net/2001/0301/a/namespaces.php3) generalize chroots. Bind mounts are how namespaces can be populated in flexible ways. See [Making a process read a different file for the same filename](https://unix.stackexchange.com/questions/81003/making-a-process-read-a-different-file-for-the-same-filename) for an example.

### Running a different distribution

Another use of chroots is to install a different distribution in a directory and run programs from it, even when they require files at hard-coded paths that are not present or have different content on the base system. This can be useful, for example, to install a 32-bit distribution on a 64-bit system that doesn't support mixed packages, to install older releases of a distribution or other distributions to test compatibility, to install a newer release to test the latest features while maintaining a stable base system, etc. See [How do I run 32-bit programs on a 64-bit Debian/Ubuntu?](https://unix.stackexchange.com/questions/12956/how-do-i-run-32-bit-programs-on-a-64-bit-debian-ubuntu) for an example on Debian/Ubuntu.

Suppose that you have an installation of your distribution's latest packages under the directory `/f/unstable`, where you run programs by switching to that directory with `chroot /f/unstable`. To make home directories available from this installations, bind mount them into the chroot:

```
mount --bind /home /f/unstable/home
```

The program [schroot](https://packages.debian.org/jessie/schroot) does this automatically.

### Accessing files hidden behind a mount point

When you mount a filesystem on a directory, this hides what is behind the directory. The files in that directory become inaccessible until the directory is unmounted. Because BSD nullfs and Linux bind mounts operate at a lower level than the mount infrastructure, a nullfs mount or a bind mount of a filesystem exposes directories that were hidden behind submounts in the original.

For example, suppose that you have a tmpfs filesystem mounted at `/tmp`. If there were files under `/tmp` when the tmpfs filesystem was created, these files may still remain, effectively inaccessible but taking up disk space. Run

```
mount --bind / /mnt
```

(Linux) or

```
mount -t nullfs / /mnt
```

(FreeBSD) to create a view of the root filesystem at `/mnt`. The directory `/mnt/tmp` is the one from the root filesystem.

### NFS exports at different paths

Some NFS servers (such as the Linux kernel NFS server before NFSv4) always advertise the actual directory location when they export a directory. That is, when a client requests `server:/requested/location`, the server serves the tree at the location `/requested/location`. It is sometimes desirable to allow clients to request `/request/location` but actually serve files under `/actual/location`. If your NFS server doesn't support serving an alternate location, you can create a bind mount for the expected request, e.g.

```
/requested/location *.localdomain(rw,async)
```

in `/etc/exports` and the following in `/etc/fstab`:

```
/actual/location /requested/location bind bind
```

### A substitute for symbolic links

Sometimes you'd like to make symbolic link to make a file `/some/where/is/my/file` appear under `/else/where`, but the application that uses `file` expands symbolic links and rejects `/some/where/is/my/file`. A bind mount can work around this: bind-mount `/some/where/is/my`to `/else/where/is/my`, and then [`realpath`](http://man7.org/linux/man-pages/man3/realpath.3.html) will report `/else/where/is/my/file` to be under `/else/where`, not under `/some/where`.

## Side effects of bind mounts

### Recursive directory traversals

If you use bind mounts, you need to take care of applications that traverse the filesystem tree recursively, such as backups and indexing (e.g. to build a [locate](http://en.wikipedia.org/wiki/Locate_(Unix)) database).

Usually, bind mounts should be excluded from recursive directory traversals, so that each directory tree is only traversed once, at the original location. With bindfs and nullfs, configure the traversal tool to ignore these filesystem types, if possible. Linux bind mounts cannot be recognized as such: the new location is equivalent to the original. With Linux bind mounts, or with tools that can only exclude paths and not filesystem types, you need to exclude the mount points for the bind mounts.

Traversals that stop at filesystem boundaries (e.g. `find -xdev`, `rsync -x`, `du -x`, …) will automatically stop when they encounter a bindfs or nullfs mount point, because that mount point is a different filesystem. With Linux bind mounts, the situation is a bit more complicated: there is a filesystem boundary only if the bind mount is grafting a different filesystem, not if it is grafting another part of the same filesystem.

## Going beyond bind mounts

Bind mounts provide a view of a directory tree at a different location. They expose the same files, possibly with different mount options and (with bindfs) different ownership and permissions. Filesystems that present an altered view of a directory tree are called _overlay filesystems_ or _stackable filesystems_. There are many other overlay filesystems that perform more advanced transformations. Here are a few common ones. If your desired use case is not covered here, check the [repository of FUSE filesystems](http://sourceforge.net/p/fuse/wiki/FileSystems/).

-   [loggedfs](http://sourceforge.net/projects/loggedfs/) — log all filesystem access for debugging or monitoring purposes ([configuration file syntax](https://unix.stackexchange.com/questions/13794/loggedfs-configuration-file-syntax/13797#13797), [Is it possible to find out what program or script created a given file?](https://unix.stackexchange.com/questions/6068/is-it-possible-to-find-out-what-program-or-script-created-a-given-file/6080#6080), [List the files accessed by a program](https://unix.stackexchange.com/questions/18844/list-the-files-accessed-by-a-program/18872#18872))

### Filter visible files

-   [clamfs](http://clamfs.sourceforge.net/) — run files through a virus scanner when they are read
    
-   [filterfs](http://filterfs.sourceforge.net/) — hide parts of a filesystem
    
-   [rofs](https://github.com/cognusion/fuse-rofs) — a read-only view. Similar to `bindfs -r`, just a little more lightweight.
    
-   [Union mounts](http://en.wikipedia.org/wiki/Union_mount) — present multiple filesystems (called _branches_) under a single directory: if `tree1` contains `foo` and `tree2` contains `bar` then their union view contains both `foo` and `bar`. New files are written to a specific branch, or to a branch chosen according to more complex rules. There are several implementations of this concept, including:
    
    -   [aufs](http://aufs.sourceforge.net/) — Linux kernel implementation, but [rejected upstream many times](https://en.wikipedia.org/wiki/Aufs)
    -   [funionfs](http://funionfs.apiou.org/?lng=en) — FUSE implementation
    -   [mhddfs](http://svn.uvw.ru/mhddfs/trunk/README) — FUSE, write files to a branch based on free space
    -   [overlay](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/Documentation/filesystems/overlayfs.txt) — Linux kernel implementation, merged upstream in Linux v3.18
    -   [unionfs-fuse](https://github.com/rpodgorny/unionfs-fuse) — FUSE, with caching and copy-on-write features

### Modify file names and metadata

-   [ciopfs](http://brain-dump.org/projects/ciopfs/) — case-insensitive filenames (can be useful to mount Windows filesystems)
-   [convmvfs](http://fuse-convmvfs.sourceforge.net/) — convert filenames between character sets ([example](https://unix.stackexchange.com/questions/67232/same-file-different-filename-due-to-encoding-problem/67273#67273))
-   [posixovl](http://sourceforge.net/projects/posixovl/) — store Unix filenames and other metadata (permissions, ownership, …) on more restricted filesystems such as VFAT ([example](https://unix.stackexchange.com/questions/108890/what-is-the-best-way-to-synchronize-files-to-a-vfat-partition/108937#108937))

### View altered file contents

-   [avfs](http://avf.sourceforge.net/) — for each archive file, present a directory with the content of the archive ([example](https://unix.stackexchange.com/questions/13749/how-do-i-recursively-grep-through-compressed-archives/13798#13798), [more examples](https://unix.stackexchange.com/search?tab=votes&q=avfs%20is%3aanswer)). There are also many [FUSE filesystems that expose specific archives as directories](http://sourceforge.net/p/fuse/wiki/ArchiveFileSystems/).
-   [fuseflt](http://users.softlab.ntua.gr/%7Ethkala/projects/fuseflt/) — run files through a pipeline when reading them, e.g. to recode text files or media files ([example](https://unix.stackexchange.com/questions/33574/how-to-use-grep-ack-with-files-in-arbitrary-encoding/33580#33580))
-   [lzopfs](https://github.com/vasi/lzopfs) — transparent decompression of compressed files
-   [mp3fs](http://khenriks.github.io/mp3fs/) — transcode FLAC files to MP3 when they are read ([example](https://unix.stackexchange.com/questions/37701/how-to-encode-huge-flac-files-into-mp3-and-other-files-like-aac/115695#115695))
-   [scriptfs](https://code.google.com/p/scriptfs/) — execute scripts to serve content (a sort of local CGI) ([example](https://unix.stackexchange.com/questions/181673/using-process-substitution-to-trick-programs-expecting-files-with-specific-exte/181680#181680))

### Modify the way content is stored

-   [chironfs](https://code.google.com/p/chironfs/) — replicate files onto multiple underlying storage ([RAID-1 at the directory tree level](https://unix.stackexchange.com/questions/14544/raid-1-lvm-at-the-level-of-directories-aka-mknodding-a-directory))
-   [copyfs](http://n0x.org/copyfs) — keep copies of all versions of the files
-   [encfs](http://www.arg0.net/encfs) — encrypt files
-   [pcachefs](https://code.google.com/p/pcachefs/) — on-disk cache layer for slow remote filesystems
-   [simplecowfs](https://github.com/vi/simplecowfs) — store changes via the provided view in memory, leaving the original files intact
-   [wayback](http://wayback.sourceforge.net/) — keep copies of all versions of the files