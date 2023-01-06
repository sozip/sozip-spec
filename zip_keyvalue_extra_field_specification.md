# Version

- Version: 0.5.0
- Date: 2023-Jan-06

# License

This specification document is (C) 2022-2023 Even Rouault and licensed under the
[CC-BY-4.0](https://spdx.org/licenses/CC-BY-4.0.html) terms.

Note: the scope of the copyrighted material does, of course, not extend onto
any source or binary code derived from the specification.

# Introduction

This document specifies a third-party extra field, to be able to encode arbitrary
key-value pairs of metadata associated with a file within a ZIP. For example
to store the Content-Type of a file.

# Specification

The encoding MUST follow the requirements of "4.6 Third Party Mappings" of the
[ZIP specification](https://pkware.cachefly.net/webdocs/APPNOTE/APPNOTE-6.3.9.TXT)

In the following, ``uint16`` is a 16-bit unsigned integer, encoded in little-endian
order (least significant byte first), and ``byte`` a 8-bit unsigned integer.

| Offset | Type   |    Name         | Comment                                    |
| ------ | ------ | --------------- | ------------------------------------------ |
|    0   | uint16 | header_id       | Constant 0x564B (bytes 'K' 'V')            |
|    2   | uint16 | payload_size    | Number of bytes following this field.      |
|    4   | byte[] | signature       | Constant "KeyValuePairs" (13 bytes)        |
|   17   | byte   | kv_count        | Number of key-value pairs.                 |
|   18   | uint16 | kv1_key_size    | Size in bytes of the first key.            |
|   20   | byte[] | kv1_key_text    | UTF-8 encoded value of the first key.      |
|        | uint16 | kv1_val_size    | Size in bytes of the first value.          |
|        | byte[] | kv1_val_text    | UTF-8 encoded value of the first value.    |
|        |        | ...             | ....                                       |
|        | uint16 | kvN_key_size    | Size in bytes of the last key.             |
|        | byte[] | kvN_key_text    | UTF-8 encoded value of the last key.       |
|        | uint16 | kvN_val_size    | Size in bytes of the last value.           |
|        | byte[] | kvN_val_text    | UTF-8 encoded value of the last value.     |

# Example

Encoding of a ("Content-Type", "text/plain") key-value pair:

| Offset  | Bytes                                  | Comment                                    |
| ------- | -------------------------------------- | ------------------------------------------ |
|    0    | 4B 56                                  | header_id                                  |
|    2    | 28 00                                  | payload_size (40 bytes)                    |
|    4    | 4B 65 79 56 61 6C 75 65 50 61 69 72 73 | signature ("KeyValuePairs")                |
|   17    | 01                                     | kv_count                                   |
|   18    | 0C 00                                  | kv1_key_size (12 bytes)                    |
|   20    | 43 6F 6E 74 65 6E 74 2D 54 79 70 65    | kv1_key_text ("Content-Type")              |
|   32    | 0A 00                                  | kv1_val_size (10 bytes)                    |
|   34    | 74 65 78 74 2F 70 6C 61 69 6E          | kv1_val_text ("text/plain")                |

Total: 44 bytes

# Annex A: Software implementations

## GDAL (C/C++ library)

The [sozip](https://github.com/rouault/gdal/tree/sozip) development branch
of [GDAL](https://gdal.org) contains an implementation of this specification.

