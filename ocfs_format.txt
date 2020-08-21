// OCFS: Ocawesome101's File System //

This filesystem was designed for a few reasons. I am fed up with trying to implement an OpenFS driver, and my other filesystem designs (MiniFS and FAT-FS) have both been FAT-based. I want a proper, relatively full-featured, UNIX filesystem from which, perhaps, to run Monolith or some such thing.

Now, the specification.

All numbers are little-endian. All references to KB, MB, TB, GB, PB, EB, etc. refer to power-of-2 numbers because I am a programmer and I don't like *iB.

// Partitioning //
What good is a filesystem without a partition on which to store it? This partitioning scheme provides a system for boot loaders and basic read-only filesystem initialization. The partition table spans sectors 2-9 of the disk and supports up to 64 partitions. A disk may be up to 8 EB in size.

+------------------------------------------------+
| Partition Table Format                         |
+-------+----------------------------------------+
| Bytes | Description                            |
+-------+----------------------------------------+
|       | Type of the filesystem the partition   |
| 1     | contains. See RESERVED PARTITION TYPES |
|       | below.                                 |
+-------+----------------------------------------+
| 2-4   | Partition flags. See PARTITION FLAGS.  |
+-------+----------------------------------------+
|       | Partition GUID - 8-byte string, each   |
| 5-12  | character ranging from 0-255. Randomly |
|       | generated on partition creation.       |
+-------+----------------------------------------+
|       | Partition label, 36 bytes. If not      |
| 13-48 | specified, should be a UUID similar to |
|       | components.                            |
+-------+----------------------------------------+
| 49-56 | Partition starting sector, 8 bytes for |
|       | future proofing.                       |
+-------+----------------------------------------+
| 57-64 | Partitioon ending sector, 8 bytes.     |
+-------+----------------------------------------+

Request a filesystem reservation and I'll add it here. Otherwise, your filesystem may not be correctly recognized!
+------------------------------------------------+
| Reserved Partition Types                       |
+-------+----------------------------------------+
| ID    | Filesystem Name                        |
+-------+----------------------------------------+
| 0x01  | OCFS                                   |
+-------+----------------------------------------+
| 0x02  | OpenFS                                 |
+-------+----------------------------------------+

These flags hold information about the partition.
+------------------------------------------------+
| Partition Flags                                |
+-------+----------------------------------------+
| Flag  | Meaning                                |
+-------+----------------------------------------+
|       |                                        |
+-------+----------------------------------------+

// Boot Loader //


/ Stage One /
The BIOS should load and execute the first 512 bytes of the disk as the stage one bootloader. If the disk is NOT bootable, this should simply display the text "Non-System Disk or Disk Error." It should wait for a keypress and restart. If the disk IS bootable, the boot loader should load and execute sectors 10 through 32 of the disk.

/ Stage Two /
The stage two boot loader should be stored between sectors 10 and 32 of the disk - a total of 11 KB of space. The rest of the disk should be dedicated to partitions.

The recommended filesystem for boot-loader data storage is the Bootloader Read-Only File System documented at https://ocfs.github.io/brofs.
