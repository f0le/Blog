adding additional volumes to openbsd vps

-add volume from hetzner cloud

confirm that disk is available

 sysctl | grep disknames
hw.disknames=cd0:,sd0:7bdaa6b67b6c4004,sd1:3192b611ced7fe68,sd2:71308e9fb14428f2,fd0:

sd1 should only show sd1:,etc

doas fdisk -iy sd0 span whole disk with mbr partition

doas disklabel /dev/sd1c , c for whole disk, see below
a add partition accept values for the, use of the whole disk, p G print for printing tables, by default uses a ? check that
q and write

check sd1 has now identifier like below:

hw.disknames=cd0:,sd0:7bdaa6b67b6c4004,sd1:3192b611ced7fe68,sd2:71308e9fb14428f2,fd0:

-create fs on partition

doas newfs sd1a unlike linux with /dev/sda

not like this:
OpenBSD$ doas newfs /dev/sd1a
newfs: /dev/sd1a: block device

mount it
OpenBSD$ doas mount /dev/sd1a /mnt

dump and restore the partition to clone

# mount /dev/sd0a /mnt
# cd /mnt
# as root:
# dump -0 -a -f - /home | restore -rf -
#
# https://wiki.ircnow.org/pmwiki.php?n=Openbsd.Newdisk
#
#

  -0-9    Dump levels.  A level 0, full backup, guarantees the entire file
             system is copied (but see also the -h option below).  A level
             number above 0, incremental backup, tells dump to copy all files
             new or modified since the last dump of a lower level.  The
             default level is 0.

     -a      "auto-size".  Bypass all tape length considerations, and enforce
             writing until an end-of-media indication is returned.  This
             option is recommended for most modern tape drives.  Use of this
             option is particularly recommended when appending to an existing
             tape, or using a tape drive with hardware compression (where you
             can never be sure about the compression ratio).

                 -f file
             Write the backup to file; file may be a special device file like
             /dev/rst0 (a tape drive), /dev/rsd1c (a disk drive), an ordinary
             file, or `-' (the standard output).  See also the TAPE
             environment variable, below.

             Multiple file names may be given as a single argument separated
             by commas.  Each file will be used for one dump volume in the
             order listed; if the dump requires more volumes than the number
             of names given, the last file name will be used for all remaining
             volumes after prompting for media changes.  If the name of the
             file is of the form "host:file" or "user@host:file", dump writes
             to the named file on the remote host using rmt(8).


             restore
              -r      Restore (rebuild) a file system.  The target file system should
             be made pristine with newfs(8), mounted, and the user changed
             working directory into the pristine file system before starting
             the restoration of the initial level 0 backup.  If the level 0
             restores successfully, the -r flag may be used to restore any
             necessary incremental backups on top of the level 0.  The -r flag
             precludes an interactive file extraction and can be detrimental
             to one's health (not to mention the disk) if not used carefully.
             An example of correct usage:

                   # newfs /dev/rsd0g
                   # mount /dev/sd0g /mnt
                   # cd /mnt
                   # restore rf /dev/rst0

             Note that restore leaves a file restoresymtable in the root
             directory to pass information between incremental restore passes.
             This file should be removed when the last incremental has been
             restored.

             restore, in conjunction with newfs(8) and dump(8), may be used to
             modify file system parameters such as size or block size.

               -f file
             Read the backup from file; file may be a special device file like
             /dev/rst0 (a tape drive), /dev/rsd1c (a disk drive), an ordinary
             file, or "-" (the standard input).  If the name of the file is of
             the form "host:file" or "user@host:file", restore reads from the
             named file on the remote host using rmt(8).

check that everything id in order:

ls -ahl

exit

doas unmount /mnt

-get unique label identifier via
$ sysctl hw.disknames
hw.disknames=wd0:bfb4775bb8397569,cd0:,wd1:56845c8da732ee7b,wd2:f18e359c8fa2522b

-add to /etc/fstab with unique identifier

reboot

doas reboot


All OpenBSD platforms use the disklabel program as the primary way to manage filesystem partitions. On the platforms that use fdisk, one MBR partition is used to hold all of the OpenBSD filesystems. This partition can be sliced into 16 disklabel partitions, labeled a through p. A few labels are special:

    a: The boot disk's a partition is your root partition.
    b: The boot disk's b partition is usually a swap partition.
    c: The c partition is always the entire disk.


clone existing partition and dump it onto the new partition

Links:
https://www.openbsd.org/faq/faq14.html

https://www.openbsdhandbook.com/disk_operations/


