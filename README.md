# Introduction

This repository contains the [specification](sozip_specification.md) for the SOZip
(Seek-Optimized Zip) profile to the ZIP file format.

[![Logo](images/logo.png)](sozip_specification.md)

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

- [![GDAL](images/gdalicon.png)](https://gdal.org): C/C++ open source geospatial library.
  Try out its [sozip branch](https://github.com/rouault/gdal/tree/sozip), pending
  future inclusion in its master branch.
- Python [sozipfile](https://github.com/sozip/sozipfile) module: drop-in replacement
  for standard ``zipfile`` module, creating SOZip-enabled files.

See [Annex A: Software implementations](https://github.com/sozip/sozip-spec/blob/master/sozip_specification.md#annex-a-software-implementations)
for more details.

# Examples of SOZip files

Examples of SOZip-enabled files can be found in the
[sozip-examples](https://github.com/sozip/sozip-examples) repository.

# Other ZIP related specification

This GitHub organization also hosts the
[KeyValuePairs extra-field specification](https://github.com/sozip/keyvaluepairs-spec/blob/master/zip_keyvalue_extra_field_specification.md),
to be able to encode arbitrary key-value pairs of metadata associated with a file
within a ZIP. For example to store the Content-Type of a file.

# Benchmarking

Done with GDAL [sozip](https://github.com/rouault/gdal/tree/sozip) branch, on a
laptop running a Intel(R) Core(TM) i7-10750H CPU @ 2.60GHz (6 cores / 12 virtual CPUs).

* ZIP generation:

| Timing | Action                                                                     |
| ------ | -------------------------------------------------------------------------- |
|  6.1 s | Multithreaded (12 vCPUs) generation of [489 MB SOZip-enabled file](https://download.osgeo.org/gdal/data/sozip/nz-building-outlines.gpkg.zip) from a 1.6 GB uncompresssed GeoPackage file with<br>``sozip nz-building-outlines.gpkg.zip nz-building-outlines.gpkg`` |
|  36 s  | Single threaded compression of same file to 480 MB with regular ``zip`` utility with<br>``zip nz-building-outlines-regular.gpkg.zip nz-building-outlines.gpkg`` |

* Bulk reading: Multithreaded extraction (4 vCPUs) of 3.2 million features with Arrow Array interface

| Timing | Action                                                                     |
| ------ | -------------------------------------------------------------------------- |
|  1.2 s | from SOZip-compressed GeoPackage file with<br>``bench_ogr_batch nz-building-outlines.gpkg.zip`` |
|  0.7 s | from uncompressed GeoPackage file with<br>``bench_ogr_batch nz-building-outlines.gpkg`` |

* Subsetting: Extraction of 66,377 features with a spatial filter

| Timing | Action                                                                     |
| ------ | -------------------------------------------------------------------------- |
|  1.2 s | from SOZip-compressed GeoPackage file with<br>``ogr2ogr out.gpkg nz-building-outlines.gpkg.zip -spat 1740000 5910000 1750000 5920000`` |
|  1.1 s | from uncompressed GeoPackage file with<br>``ogr2ogr out.gpkg nz-building-outlines.gpkg -spat 1740000 5910000 1750000 5920000`` |

* Extraction of one feature from its identifier:

| Timing | Action                                                                     |
| ------ | -------------------------------------------------------------------------- |
|  45 ms | from SOZip-compressed GeoPackage file with<br>``ogr2ogr out.gpkg nz-building-outlines.gpkg.zip -fid 1000000`` |
|  44 ms | from uncompressed GeoPackage file with<br>``ogr2ogr out.gpkg nz-building-outlines.gpkg -fid 1000000`` |

# How to contribute ?

We welcome contributions to this specification as [issues](https://github.com/sozip/sozip-spec/issues),
[pull requests](https://github.com/sozip/sozip-spec/pulls) or
[discussions](https://github.com/sozip/sozip-spec/discussions).

If you use SOZip or plan to use it for your data delivery, or consider doing a
SOZip implementation, etc., let us know!

# Datasets available as SOZip

* [TUDelft3D](https://3d.bk.tudelft.nl/) delivers the [3DBAG](https://docs.3dbag.nl/en/) data, an open 3D building dataset, for the whole of the Netherlands, 108 GB uncompressed, as a 18.8 GB large GeoPackage: https://3dbag.nl/en/download

# Social media

Find me on [![Mastodon](images/Mastodon_Logotype_(Simple).png)](https://fosstodon.org/@sozip)

# Credits

The SOZip specification and its GDAL implementation have been developed by
[Spatialys](https://www.spatialys.com), with support from [Safe Software](https://www.safe.com/)

<!---
# Adopters

(Put here a list of organizations, in particular data producers, that have
adopted SOZip)
-->
