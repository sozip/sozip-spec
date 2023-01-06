# Introduction

This repository contains the [specification](sozip_specification.md) for the SOZip
(Seek-Optimized Zip) profile to the ZIP file format.

![Logo](images/logo.png)

# What is SOZip ?

A Seek-Optimized ZIP file (SOZip) is a
[ZIP](https://en.wikipedia.org/wiki/ZIP_(file_format)) file that contains one
or several [Deflate](https://www.ietf.org/rfc/rfc1951.txt)-compressed files
that are organized and annotated such that a SOZip-aware reader can perform
very fast random access (seek) within a compressed file.

SOZip makes it possible to access large compressed files directly from a .zip
file without prior decompression. It is *not* a new file format, but a profile
of the existing ZIP format, done in a fully backward compatible way. ZIP
readers that are non-SOZip aware can read a SOZip-enabled file
normally and ignore the extended features that support efficient seek
capability.

# Software implementations

- C/C++ [GDAL](https://gdal.org)
- Python [sozipfile](https://github.com/sozip/sozipfile)

See [Annex A: Software implementations](https://github.com/sozip/sozip-spec/blob/master/sozip_specification.md#annex-a-software-implementations)
for more details.

# Examples of SOZip files

Examples of SOZip-enabled files can be found in the
[sozip-examples](https://github.com/sozip/sozip-examples) repository.

# Other ZIP related specification

This repository also contains the [KeyValuePairs extra-field specification](zip_keyvalue_extra_field_specification.md),
to be able to encode arbitrary key-value pairs of metadata associated with a file
within a ZIP. For example to store the Content-Type of a file.


# Social media

Find me on [![Twitter](images/32px-Twitter-logo.svg.png)](https://twitter.com/sozipOrg) and
[![Mastodon](images/Mastodon_Logotype_(Simple).png)](https://fosstodon.org/@sozip)

# Credits

The SOZip specification and its GDAL implementation have been developed by
[Spatialys](https://spatialys.com), with support from [Safe Software](https://www.safe.com/)

<!---
# Adopters

(Put here a list of organizations, in particular data producers, that have
adopted SOZip)
-->
