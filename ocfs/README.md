# // OCFS: Ocawesome101's File System //

This filesystem was designed for a few reasons. I am fed up with trying to implement an OpenFS driver, and my other filesystem designs (MiniFS and FAT-FS) have both been FAT-based. I want a proper, relatively full-featured, UNIX filesystem from which, perhaps, to run Monolith or some such thing.

Now, the specification.

All numbers are little-endian. All references to KB, MB, TB, GB, PB, EB, etc. refer to power-of-2 numbers because I am a programmer and I don't like \*iB.

## // The Filesystem //

OCFS uses inodes, sometimes known as index nodes, for file storage. All sectors excepting inodes and data sectors are unused. All references to "sector" are relative to a partition.

Sectors should be marked as used or unused - if the first byte of a sector is the ASCII character `i`, the sector is an inode. If the first byte is the ASCII character `d`, the sector contains file data. If the first byte is the ASCII character `e`, the sector contains extended file sector data. If the first byte is the ASCII null character `\0`, the sector is unused. These flags must be set when creating and deleting files.

Sector 1 holds some information about the filesystem. The root inode is stored on sector 2.

```
+------------------------------------------------+
| OCFS Superblock Format                         |
+--------+---------------------------------------+
| Bytes  | Description                           |
+--------+---------------------------------------+
| 1-7    | Filesystem signature: ASCII string    |
|        | '\27OCFS\13\27'                       |
+--------+---------------------------------------+
| 8      | Filesystem flags - see 'Filesystem    |
|        | Flags' below.                         |
+--------+---------------------------------------+
| 9-512  | (8-byte chunks) Free sector cache.    |
+--------+---------------------------------------+

+------------------------------------------------+
| Filesystem Flags                               |
+--------+---------------------------------------+
| Flag   | Meaning                               |
+--------+---------------------------------------+
| 0x01   | Whether the filesystem is read-only.  |
+--------+---------------------------------------+
| 0x02   | Whether the filesystem is encrypted.  |
|        | Encryption uses AES-256.              |
+--------+---------------------------------------+

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
| 0x01      | Owner R    |
+-----------+------------+
| 0x02      | Owner W    |
+-----------+------------+
| 0x04      | Owner X    |
+-----------+------------+
| 0x08      | Group R    |
+-----------+------------+
| 0x10      | Group W    |
+-----------+------------+
| 0x20      | Group X    |
+-----------+------------+
| 0x40      | Other R    |
+-----------+------------+
| 0x80      | Other W    |
+-----------+------------+
| 0x100     | Other X    |
+-----------+------------+
```
