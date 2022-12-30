# Introduction

This repository contains the [specification](spec.md) for the SOZip
(Seek-Optimized Zip) profile to the ZIP file format.

![Logo](logo.png)

# What is SOZip ?

A Seek-Optimized ZIP file (SOZip) is a [ZIP](https://en.wikipedia.org/wiki/ZIP_(file_format)) file
that contains one or several Deflate-compressed files, optimized such that a
SOZip-aware reader can perform very fast random access (seek) within a
compressed file.

This opens the possibility of using large compressed files directly from a
.zip file, without prior decompression.

SOZip is *not* a new file format, but a profile of the existing ZIP format,
done in a fully backward compatible way. ZIP readers that are non-SOZip aware
should be able to read a SOZip-enabled file, ignoring the extended features.

# Software implementations

- C/C++ [GDAL](https://gdal.org)
- Python [sozipfile](https://github.com/sozip/sozipfile)

See [Annex A: Software implementations](https://github.com/sozip/sozip-spec/blob/master/spec.md#annex-a-software-implementations)
for more details.

# Examples of SOZip files

Examples of SOZip-enabled files can be found in the
[sozip-examples](https://github.com/sozip/sozip-examples) repository.

<!---
# Adopters

(Put here a list of organizations, in particular data producers, that have
adopted SOZip)
-->
