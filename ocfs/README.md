# // OCFS: Ocawesome101's File System //

This filesystem was designed for a few reasons. I am fed up with trying to implement an OpenFS driver, and my other filesystem designs (MiniFS and FAT-FS) have both been FAT-based. I want a proper, relatively full-featured, UNIX filesystem from which, perhaps, to run Monolith or some such thing.

Now, the specification.

All numbers are little-endian. All references to KB, MB, TB, GB, PB, EB, etc. refer to power-of-2 numbers because I am a programmer and I don't like \*iB.

### // Partitioning //
What good is a filesystem without a partition on which to store it? The OCGPT (for OpenComputers General Partition Table) partition table spans sectors 2-9 of the disk and supports up to 64 partitions. A disk may be up to 8 EB in size.

```
+------------------------------------------------+
| OCGPT Partition Table Format                   |
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
| 0x03  | FoxFS                                  |
+-------+----------------------------------------+
| 0x04  | ZebraFS                                |
+-------+----------------------------------------+
| 0x05  | NitroFS                                |
+-------+----------------------------------------+
| 0x06  | BROFS                                  |
+-------+----------------------------------------+
```

These flags hold information about the partition.
```
+------------------------------------------------+
| Partition Flags                                |
+-------+----------------------------------------+
| Flag  | Meaning                                |
+-------+----------------------------------------+
| 0x01  | The partition is bootable.             |
+-------+----------------------------------------+
| 0x02  | The partition should emulate a managed |
|       | filesystem.                            |
+-------+----------------------------------------+
| 0x04  | The BIOS, if following OCUEFI, should  |
|       | load bootloader.lua from this          |
|       | partition. Only guaranteed to be valid |
|       | for BROFS as many other filesystems    |
|       | are too complex to reasonably fit their|
|       | drivers into an EEPROM.                |
+-------+----------------------------------------+
```

#### // Boot Loader //

##### / Stage One /
The BIOS should load and execute the first 512 bytes of the disk as the stage one bootloader. If the disk is NOT bootable, this should simply display the text `Non-System Disk or Disk Error.`, wait for a keypress, and restart. If the disk IS bootable, the boot loader should load and execute sectors 10 through 32 of the disk.

##### / Stage Two /
The stage two boot loader should be stored between sectors 10 and 32 of the disk - a total of 11 KB of space. The rest of the disk should be dedicated to partitions.

The recommended filesystem for storage of bootloader data, such as filesystem drivers, is the Bootloader Read-Only File System documented [here](https://ocfs.github.io/brofs).

#### // Similar to EFI: OpenComputers Unmanaged Extensible Firmware Interface //
OCGPT provides a method for booting disks using a method similar to EFI: OCUEFI, or the OpenComputers Unmanaged Extensible Firmware Interface.

Given the simplicity of BROFS, a basic read-only driver can be implemented in a very small space such as an EEPROM. The BIOS stored in EEPROM may load a boot script as `bootloader.lua` from the BROFS partition to act as the bootloader, rather than the bootloader being stored as raw data on the disk.

##### / OCUEFI bootloaders /

OCUEFI bootloaders differ slightly from BIOS-style bootloaders.

On initialization, an OCUEFI bootloader should parse the OCGPT and load any necessary drivers from the BROFS partition. If applicable, the bootloader should present a menu for selection of a boot device; if not applicable, the bootloader should either proceed to boot from the configured partition, display a panic message, or present a command line.

## // The Filesystem //

OCFS uses inodes, sometimes known as index nodes, for file storage. All sectors excepting inodes and data sectors are unused.

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
|        | in order. Any excess should set to 0. |
+--------+---------------------------------------+
|505-512 | Pointer to a sector containing        |
|        | extensions of the previous pointer    |
|        | section; value should always be 0 if  |
|        | there is none.                        |
+--------+---------------------------------------+

+------------------------------------------------+
| File Types                                     |
+--------+---------------------------------------+
| Byte   | Meaning                               |
+--------+---------------------------------------+
| 0x00   | Invalid inode.                        |
+--------+---------------------------------------+
| 0x01   | Inode describes a text document, or   |
|        | file.                                 |
+--------+---------------------------------------+
| 0x02   | Inode describes a directory.          |
+--------+---------------------------------------+
| [TODO] | Add support for file linking          |
+--------+---------------------------------------+
```

### / File Permissions /

OCFS file permissions are laid out over 9 bits (the final 7 bits of the allocated two bytes are unused). If a bit is set, then the corresponding permission should be considered set.

```
+------------------------+
| BIT LAYOUT             |
-------------------------+
| Bit       | Permission |
+-----------+------------+
| 100000000 | Owner R    |
+-----------+------------+
| 010000000 | Owner W    |
+-----------+------------+
| 001000000 | Owner X    |
+-----------+------------+
| 000100000 | Group R    |
+-----------+------------+
| 000010000 | Group W    |
+-----------+------------+
| 000001000 | Group X    |
+-----------+------------+
| 000000100 | Other R    |
+-----------+------------+
| 000000010 | Other W    |
+-----------+------------+
| 000000001 | Other X    |
+-----------+------------+
```
