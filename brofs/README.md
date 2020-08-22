# BROFS - Bootloader Read-Only FileSystem

BROFS is a simple filesystem usable by bootloaders to load other filesystem drivers. It does not support directories, is limited to 64 files, and filenames are limited to 24 characters, making it in effect an organized pile of data with filenames on top.

BROFS's fixed-size file table occupies the first 1024 bytes (2 sectors) of a partition. On boot, this file table is loaded into memory and parsed.

Each inode is 32 bytes:
```
+------------------------------------------------+
| BROFS File Table Entry Format                  |
+-------+----------------------------------------+
| Bytes | Description                            |
+-------+----------------------------------------+
| 1-2   | Starting data sector                   |
+-------+----------------------------------------+
| 3-4   | Size in bytes, rounded up to sectors.  |
+-------+----------------------------------------+
| 5-6   | Preallocated file data space.          |
+-------+----------------------------------------+
| 7     | Flags - entry is only valid if >=1     |
+-------+----------------------------------------+
| 8     | Unused.                                |
+-------+----------------------------------------+
| 9-32  | Filename (24 bytes)                    |
+-------+----------------------------------------+
```
