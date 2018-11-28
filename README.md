# rhel-utilities

# Quick and dirty guide to creating and using multipathed volumes


## References:

[RedHat Guide to DM Multipath] (http://www.redhat.com/docs/en-US/Red_Hat_Enterprise_Linux/5.4/html/DM_Multipath/index.html)

[RedHat Guide to LVM Admin] (http://www.redhat.com/docs/en-US/Red_Hat_Enterprise_Linux/5.4/html/Logical_Volume_Manager_Administration/index.html)

[Linuxtopia RHEL5 Admin Guide] (http://www.linuxtopia.org/online_books/rhel5/rhel5_administration/index.html)

[RedHat RHEL5 Library] (http://www.redhat.com/docs/manuals/enterprise/#RHEL5)

## The extremely short version

Here's what you need to do:

1. Carve and present your LUNs from the SAN
2. Set them up with multipathing
3. Define each mapped device as a physical volume
4. Carve LVs from the PVs defined in 3. and build filesystems if necessary

It is assumed that the miracle identified in 1. has already occurred.

To set up multipathing, you need to create an appropriate multipath configuration file (`/etc/multipath.conf`)
and relaunch the multipath daemon.  (Note: make sure you have the multipath RPM installed before you try this.
You can check this by running rpm -qa | grep multipath.  If you get a result, you're good.  If you don't, you
need to stop and install the device-mapper-multipath package before going further.)

The DM Multipath guide above provides detail on the structure and syntax of multipath.conf.  For convenience,
a multipath.conf generator utility has been written, named multipath-conf-generator, which will generate a
file suitable for use with LUNs presented from 3PAR arrays.  It expects a filename on the command line, which
contains details of the presented LUNs.  the format of that file is `alias size WWID`:

```
db_data_a1      64      50002ac0005b0854
db_data_a2      128     50002ac0005c0854
```

The size attribute is not used but it is helpful when you are double-checking the LUN aliasing.

Once the LUNs have been presented, you should see your aliases showing up under `/dev/mapper` like so:

```
brw-rw---- 1 root disk 253,  8 Oct 16 15:40 db_index_b1
brw-rw---- 1 root disk 253,  9 Oct 16 15:40 db_index_b2
brw-rw---- 1 root disk 253, 10 Oct 16 15:40 db_index_x
brw-rw---- 1 root disk 253, 11 Oct 16 15:40 db_redo_a
brw-rw---- 1 root disk 253, 12 Oct 16 15:40 db_redo_b
```

You will also see entries showing up under `/dev/mpath` and `/dev/dm-x`, where x is a number.
DO NOT USE THESE ENTRIES in scripting or other processes, since the are not intended for anything other than
internal use and may not even be visible until the boot process has completed.

To label the newly-presented and mapped LUNs as physical devices, you need to run pvcreate.  This will initialize
them and make them available to the logical volume manager.  The form of the command is:

```
pvcreate -v -M2 /dev/mapper/<alias>
```

where -v provides verbose output and -M2 uses the LVM2 format for storing metadata on the volume.

A helper script called `make-pvs` has been provided, which uses the same input file format as
`multipath-conf-generator`.  An example of its output is as follows:

```
Making PV for device /dev/mapper/scratch ...
    Set up physical volume for "/dev/mapper/scratch" with 41943040 available sectors
    Zeroing start of device /dev/mapper/scratch
  Physical volume "/dev/mapper/scratch" successfully created

Making PV for device /dev/mapper/backup ...
    Wiping cache of LVM-capable devices
    Set up physical volume for "/dev/mapper/backup" with 2147483648 available sectors
    Zeroing start of device /dev/mapper/backup
  Physical volume "/dev/mapper/backup" successfully created
```


Once the physical volumes have been created, they can be added to volume groups and used for logical volume
allocation, just like any other block device.

An example script, named `make-lvs-and-filesystems` has been provided as part of this package for illustration.

Enjoy!!