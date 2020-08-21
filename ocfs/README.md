# // OCFS: Ocawesome101's File System //

This filesystem was designed for a few reasons. I am fed up with trying to implement an OpenFS driver, and my other filesystem designs (MiniFS and FAT-FS) have both been FAT-based. I want a proper, relatively full-featured, UNIX filesystem from which, perhaps, to run Monolith or some such thing.

Now, the specification.

All numbers are little-endian. All references to KB, MB, TB, GB, PB, EB, etc. refer to power-of-2 numbers because I am a programmer and I don't like \*iB.

### // Partitioning //
What good is a filesystem without a partition on which to store it? This partitioning scheme provides a system for boot loaders and basic read-only filesystem initialization. The partition table spans sectors 2-9 of the disk and supports up to 64 partitions. A disk may be up to 8 EB in size.

```
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
```

Request a filesystem reservation and I'll add it here. Otherwise, your filesystem may not be correctly recognized!
```
+------------------------------------------------+
| Reserved Partition Types                       |
+-------+----------------------------------------+
| ID    | Filesystem Name                        |
+-------+----------------------------------------+
| 0x01  | OCFS                                   |
+-------+----------------------------------------+
| 0x02  | OpenFS                                 |
+-------+----------------------------------------+
```

These flags hold information about the partition.
```
+------------------------------------------------+
| Partition Flags                                |
+-------+----------------------------------------+
| Flag  | Meaning                                |
+-------+----------------------------------------+
|       |                                        |
+-------+----------------------------------------+
```

#### // Boot Loader //

##### / Stage One /
The BIOS should load and execute the first 512 bytes of the disk as the stage one bootloader. If the disk is NOT bootable, this should simply display the text `Non-System Disk or Disk Error.`, wait for a keypress, and restart. If the disk IS bootable, the boot loader should load and execute sectors 10 through 32 of the disk.

##### / Stage Two /
The stage two boot loader should be stored between sectors 10 and 32 of the disk - a total of 11 KB of space. The rest of the disk should be dedicated to partitions.

The recommended filesystem for storage of bootloader data, such as filesystem drivers, is the Bootloader Read-Only File System documented [here](https://ocfs.github.io/brofs).

## // The Filesystem //

Now, OCFS documentation.

OCFS uses inodes, or index nodes, for file storage. All sectors excepting inodes and data sectors are unused.

Sectors should be marked as used or unused - if the first byte of a sector is the ASCII character `i`, the sector is an inode. If the first byte is the ASCII character `d`, the sector contains file data. If the first byte is the ASCII character `e`, the sector contains extended file sector data. If the first byte is the ASCII null character `\\0`, the sector is unused. These flags must be set when creating and deleting files.

```
+------------------------------------------------+
| Inode Format                                   |
+--------+---------------------------------------+
| Bytes  | Description                           |
+--------+---------------------------------------+
| 1      | ASCII character 'i'; inode signature. |
+--------+---------------------------------------+
| 2      | File type. See 'File Types' below.    |
+--------+---------------------------------------+
| 3-4    | File permissions, 2 bytes. See 'File  |
|        | Permissions' below.                   |
+--------+---------------------------------------+
| 5-8    | Owner UID, 4 bytes.                   |
+--------+---------------------------------------+
| 9-12   | Group ID, 4 bytes.                    |
+--------+---------------------------------------+
| 11-20  | POSIX 64-bit last modified timestamp. |
+--------+---------------------------------------+
| 21-64  | Case-sensitive filename, up to 44     |
|        | ASCII characters or bytes.            |
+--------+---------------------------------------+
| 65-504 | 8-byte chunks: If the inode describes |
|        | a file, pointers to the file data     |
|        | sectors, in order. If the inode       |
|        | describes a directory, contains       |
|        | pointers to the directory's children, |
|        | in order.                             |
+--------+---------------------------------------+
|505-512 | Pointer to a sector containing        |
|        | extensions of the previous pointer    |
|        | section; value should always be 0 if  |
|        | there is none.                        |
+--------+---------------------------------------+
```
