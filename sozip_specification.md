# License

This specification document is (C) 2022-2023 Even Rouault and licensed under the
[CC-BY-4.0](https://spdx.org/licenses/CC-BY-4.0.html) terms.

Note: the scope of the copyrighted material does, of course, not extend onto
any source or binary code derived from the specification, that may be licensed
under the terms that their author sees fit.

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

# Use cases

This specification is intended to be general purpose / not domain specific.

Nevertheless it has been developed to serve a few geospatial use cases. In
particular, to make it possible for users to read (large) files using the
Shapefile, GeoPackage or FlatGeobuf formats (which have no native provision for
compression) compressed in .zip files, in Geographic Information Systems (GIS)
without prior decompression.

Efficient random access to features within such files is a requirement to get
acceptable performance in usage scenarios: spatial index filtering, access to a
feature by its identifier, etc.


# High-level specification

The SOZip optimization relies on two independant and combined mechanisms:

* the first one is the generation of a [Deflate](https://www.ietf.org/rfc/rfc1951.txt)
  compressed stream which is structured in a way such that it contains chunks of
  data that can be compressed and uncompressed in an independant way from
  preceding and following chunks in the compressed stream. Conceptually, such
  a compressed file could be seen as split in multiple independantly compressed
  files. But from the point of view of a non-SOZip-aware ZIP reader, this will
  still be a fully single legit compressed stream for the whole file.
  That chunking relies on the block flush mechanisms of the ZLib library, which is
  typically used by the [pigz](https://zlib.net/pigz/pigz.pdf) utility with its
  *--independant* option. Those block flushes are done at a regular interval of
  the input uncompressed stream. In the rest of this document, this interval is
  called chunk size. A typical value for it is 32 kilobytes.

* the creation of a hidden index file which contains an array which maps file
  offsets of the uncompressed file, at every chunk size interval, to the
  corresponding offset in the Deflate compressed stream.


# Detailed specification

A ZIP file is said to be SOZip-enabled if it contains one or several Deflate
compressed files meeting the following requirements, in additions to the
requirements of the [.ZIP File Format Specification](https://pkware.cachefly.net/webdocs/APPNOTE/APPNOTE-6.3.9.TXT).

A SOZip-enabled file may contain a mix of SOZip-compressed and regular compressed
or uncompressed files.

A file may be SOZIP-compressed only if its uncompressed size is strictly greater
than the chunk size  (otherwise there is no point in doing the SOZip optimization).

## Chunked Deflate-compressed stream

A SOZip-compressed file MUST be created with compression_method = 8 (Deflate).

A SOZip-compressed file MUST have a corresponding
[local file header](https://en.wikipedia.org/wiki/ZIP_(file_format)#Local_file_header)
and a [central directory file header](https://en.wikipedia.org/wiki/ZIP_(file_format)#Central_directory_file_header).

Those headers MAY use extended fields. Typically for the
[ZIP64](https://en.wikipedia.org/wiki/ZIP_(file_format)#ZIP64) extension if
the compressed and/or uncompressed size of a file exceeds 4 GB. Or the UTF-8
file name extension.

A SOZip writer MUST issue a call to ``deflate()``
[ZLib](https://www.zlib.net/manual.html) method with the ``Z_SYNC_FLUSH`` mode,
followed by a call with the ``Z_FULL_FLUSH`` flag, at a fixed interval (called
chunk size) of the data read from the input uncompressed stream.

``Z_SYNC_FLUSH`` and ``Z_FULL_FLUSH`` are not required (but may be used) for
the final chunk, whose size may be smaller or equal to the chunk size.
However the last call to ``deflate()`` to encode the last chunk MUST be done
with the ``Z_FINISH`` flag, to finalize a valid Deflate stream.

Note: an explanation of the ``Z_SYNC_FLUSH`` and ``Z_FULL_FLUSH`` mode can be found at
https://www.bolet.org/~pornin/deflate-flush-fr.html

The writer MUST collect the offset (relative to the start of the compressed
data) of each chunk, except for the initial chunk size whose offset is zero.

Note: a pseudo-code (among many possible variations) written in C++, using
zlib and assuming offset_size == 8 (cf later paragraph), can be found in
[Annex E](#annex-e-pseudo-code-for-sozip-deflate-stream-generation)

## Hidden index file

### Storage of the index file

The index file MUST be stored as a uncompressed file.

The index file name MUST be :
- "${path_to_filename}/.${filename}.sozip.idx" where ${path_to_filename} is the name
  of the directory if ${filename} contains directory paths.
  For example "my_dir/.rivers.gpkg.sozip.idx" if the filename stored in the
  archive is "my_dir/rivers.gpkg.sozip.idx"
- or ".${filename}.sozip.idx" if there is no directory path in the filename.
  For example ".rivers.gpkg.sozip.idx"

Note the leading dot ('.') character preceding the index filename, to indicate
its hidden status.

The index file MUST be preceded by a ZIP local file header (cf paragraph 4.3.7
of the [.ZIP File Format Specification](https://pkware.cachefly.net/webdocs/APPNOTE/APPNOTE-6.3.9.TXT))

That local file header MAY use extended fields. Typically for the UTF-8
file name extension.

That local file header MUST be put immediately placed after the compressed file.

That file MUST NOT be listed as a central file header, to remain invisible.

### Content of the index file

The hidden index file is made of a 32-byte header followed by a section of
varying size (called offset section) which contains the values of the offsets
collected during the generation of the Deflate-compressed data.

In the following, ``uint32`` is a 32-bit unsigned integer, encoded in little-endian
order (least significant byte first).
``uint64`` is a 64-bit unsigned integer, encoded in little-endian order.

#### Header

| Offset | Type   |    Name         | Comment                                    |
| ------ | ------ | --------------- | ------------------------------------------ |
|    0   | uint32 | version         | Version number.                            |
|    4   | uint32 | skip_bytes      | Number of bytes to skip after header.      |
|    8   | uint32 | chunk_size      | Chunk size in bytes.                       |
|   12   | uint32 | offset_size     | Size in bytes of an entry in offset section|
|   16   | uint64 | uncompress_size | Size in bytes of the uncompressed file.    |
|   24   | uint64 | compress_size   | Size in bytes of the compressed file.      |

Specification of fields:

* ``version``: MUST be set to 1 for this specification

* ``skip_bytes``: number of bytes between the end of the header and the beginning
  of the offset section. Generally set to 0.

* ``chunk_size``: Interval, in uncompressed stream, at which Z_SYNC_FLUSH +
  Z_FULL_FLUSH are performed. It MUST be strictly greater than zero (a value
  of 4096 or bigger is strongly recommended). A value lower than 100 MB
  is strongly recommended for performance and compatibility with SOZip readers.
  32 KB is a generally safe default value.

* ``offset_size``: Number of bytes in which offsets in the offset section are
  recommended. The only two supported values are 4 (uint32) and 8 (uint64).
  A writer MUST use 8 if any value in the offset section can not fit in a uint32
  integer.

* ``uncompress_size``: Size in bytes of the uncompressed file (not the index, but
  the file subject to SOZip compression). This field is redundant with other
  information found in the local and central file headers of the compressed file,
  and is here so that a reader can check the consistency of the SOZip index
  with the compressed file. ``uncompress_size`` must be strictly greater than
  ``chunk_size``

* ``compress_size``: Size in bytes of the compressed file (not the index, but
  the file subject to SOZip compression). This field is redundant with other
  information found in the local and central file headers of the compressed file,
  and is here so that a reader can check the consistency of the SOZip index
  with the compressed file.

#### Offset section

The offset section MUST contain exactly ``(uncompress_size - 1) / chunk_size``
(floor rounding) entries, each of size ``offset_size`` bytes.

Each entry is a uint32 (if ``offset_size`` = 4) or a uint64 (if ``offset_size`` = 8)
expressing the offset in the uncompressed stream at which a compressed chunk
starts. The offset of the first compressed chunk is omitted, as always 0.

The first offset value is thus the offset in the compressed stream where
uncompressed bytes in the range
``[chunk_size, min(2 * chunk_size, uncompressed_size)[`` are encoded.

The second offset value is the offset in the compressed stream where
uncompressed bytes in the range
``[2 * chunk_size, min(3 * chunk_size, uncompressed_size)[`` are encoded. And so on.

As a consequence of the generation of the index, values in the offset section
MUST be in strictly ascending order, and MUST be stricly lower than
``compress_size``.

# Normative references

- [.ZIP File Format Specification](https://pkware.cachefly.net/webdocs/APPNOTE/APPNOTE-6.3.9.TXT)

- [DEFLATE Compressed Data Format Specification version 1.3](https://www.ietf.org/rfc/rfc1951.txt)

# Annex A: Software implementations

## GDAL (C/C++ library)

The [sozip](https://github.com/rouault/gdal/tree/sozip) development branch
of [GDAL](https://gdal.org) contains:

* a ``CPLAddFileInZip()`` C function that can compress a file and add it to an
  new or existing ZIP file, and enable the SOZip optimization when relevant.

* an implementation of the
  [VSIGetFileMetadata()](https://gdal.org/api/cpl.html#_CPPv418VSIGetFileMetadataPKcPKc12CSLConstList)
  method that can be used on a filename of the form "/vsizip//path/to/my.zip/filename/inside" and with
  domain = "ZIP" to get information if a SOZip index is available for that file.

* a modified [/vsizip/](https://gdal.org/user/virtual_file_systems.html#vsizip-zip-archives)
  virtual file system that can use the SOZip optimization to perform fast random
  access to a compressed file within a ZIP.

* a new command line utility,
  [sozip](https://github.com/rouault/gdal/blob/sozip/doc/source/programs/sozip.rst),
  that can be used to create a seek-optimized ZIP file, to append files to an
  existing ZIP file, list the contents of a ZIP file and display the SOZip
  optimization status or validate a SOZip file.

* Updated Shapefile and GeoPackage drivers that can directly generate SOZip-enabled
  .shz/.shp.zip or .gpkg.zip files.

This development branch is available in the ``rouault/sozip`` Docker image.

Examples:

* Create a new ZIP file with an input file called in.gpkg:

  ```shell
  docker run --rm -it -v $PWD:$PWD rouault/sozip sozip -j $PWD/out.zip $PWD/in.gpkg
  ```

* Create a SOZip-optimized zip file called out.zip from an existing ZIP file
  called in.zip.

  ```shell
  docker run --rm -it -v $PWD:$PWD rouault/sozip sozip --convert-from=$PWD/in.zip $PWD/out.zip
  ```

* List the content of a ZIP file and check if files in it are SOZip-optimized:

  ```shell
  docker run --rm -it -v $PWD:$PWD rouault/sozip sozip -l $PWD/in.zip
  ```

* Validate a SOZip file:

  ```shell
  docker run --rm -it -v $PWD:$PWD rouault/sozip sozip --validate $PWD/my.zip
  ```

## sozipfile (Python module)

[sozipfile](https://github.com/sozip/sozipfile) is a fork of Python
[zipfile](https://docs.python.org/3/library/zipfile.html) module, which
implements the SOZip implementation in the ZIP writer.

## MapServer (Web mapping server written in C/C++, using GDAL)

The [sozip](https://github.com/rouault/mapserver/tree/sozip) development branch
of [MapServer](https://mapserver.org), when built against a SOZip-capable GDAL,
can generate SOZip-enabled output files if the mapfile has a ZIP output format,
such as:

```
    OUTPUTFORMAT
      NAME "OGRGPKGZIP"
      DRIVER "OGR/GPKG"
      MIMETYPE "application/zip; driver=ogr/gpkg"
      FORMATOPTION "STORAGE=memory"
      FORMATOPTION "FORM=zip"
      FORMATOPTION "FILENAME=result.gpkg.zip"
    END
```

## QGIS (Geographic Information System desktop and server application, using GDAL)

[QGIS](https://qgis.org) can read efficiently SOZip files when built against a
SOZip-capable GDAL, through the use of GDAL ``/vsizip/`` virtual file system.


# Annex B: Advantages and limitations

## Advantages

* SOZip allows multithreaded compression of independant chunks. This is for
  example used in the GDAL implementation.

* SOZip allows multithreaded decompression of independant chunks.

* For decompression, faster alternatives to zlib can be used, such as
  [libdeflate](https://github.com/ebiggers/libdeflate). This is for example used
  in the GDAL implementation.

* SOZip has excellent backward compatibility. A data producer may deliver a
  SOZip enabled file with good confidence that nearly all existing ZIP readers
  can decompress it (at time of writing, we are not aware of ZIP readers that
  reject a SOZip enabled file.)

## Limitations

* Compression efficiency is reduced by the flushes done to isolate chunks.
  The larger the chunk size, the more efficient the compression, but
  random seeking will be less efficient due to more data being decompressed.

* SOZip inherits all the limitations of the base ZIP format: in particular
  update in place of a SOZip optimized file requires rewriting the entire ZIP,
  or appending the updated version of the modified file at the end of the ZIP
  (with rewriting of the central header records and end of central directory
  record).

# Annex C: Discussion about design choices

* Why use the Deflate compression and not an alternative compression method ?

  Deflate has been chosen as it is supported by all existing ZIP implementations.
  Other compression methods (LZMA, BZIP2, etc.) are supported more sparsely.
  Furthermore, given that the SOZip optimization results in non-optimal
  compression rate, it is likely that compression schemes that offer higher
  compression than Deflate would perform in a suboptimal way, due to the chunking
  mechanism resetting the dictionary at each chunk boundary.

* Why encoding the hidden index as hidden and not visible ?

  This design decision has pros and cons.

  Pros:
  - End-users will not see those indexes which are not directly useful for them.
  - Non SOZip-aware software will not be disturbed by a file they don't expect
    (some readers could for example expect a precise list of files to be in a
    .zip)
  - If the index file was visible, and the SOZip archive was regenerated by
    a non SOZip-aware writer that does an edit operation to an existing archive
    by recreating a new file, it would preserve the index file, but could
    potentially recompress the compressed file without using the chunked Deflate
    techniques, which could confuse SOZip readers (although a SOZip reader must
    use information from the header of the SOZip index to check its consistence
    with the compressed file).

  Cons:
  - Appending new files to a SOZip-enabled file may cause the SOZip index
    to be lost when using some non SOZip-aware ZIP writer. However, ZIP
    implementations that have an append-in-place strategy will generally
    preserve the hidden index. Refer to
    [Annex D](#annex-d-compatibility-with-existing-zip-implementations) for a
    list of known implementations that can append-in-place.

* Why encoding the hidden index as a file preceded by a local header ?

  We have found at least one reader, Java's
  [java.util.zip.ZipInputStream](https://docs.oracle.com/javase/8/docs/api/java/util/zip/ZipInputStream.html),
  which operates in a streaming way, and stops its enumeration of the content of
  a ZIP file at the first encountered content that is not a local header.

* Why placing the hidden index after the compressed file and not before ?

  Both can make sense. It has been observed though that if a hidden local header
  (that is not listed in the central directory entries) is located immediately
  at the beginning of a zip file, the ``7zip`` utility will expose the hidden file.
  And, combined with the question at the previous paragraph, if the content
  is not preceded by a local header, ``7zip`` will emit a warning
  ("The archive is open with offset").
  It is also slightly easier to generate the index after the compressed file,
  given that the content of the index depends on information collected during
  creation of the chunked Deflate compressed stream. This also makes it
  potentially possible for a streaming writer to write a SOZip optimized file.

* Why not compressing the index file ?

  Having it uncompressed makes it easier for implementations. And for very large
  files, having it uncompressed makes it possible to seek at a random location
  in a truly constant time. A 64 GB compressed file, with a chunk size of 32 KB,
  requires a 15.6 MB index. For scenarios where a SOZip file is read in a
  on-demand piece-wise way from network storage, it would be costly in bandwith
  to have to download and decompress those 15 MB to read the last chunk of the
  64 GB compressed file.


# Annex D: Compatibility with existing ZIP implementations

SOZip-enabled files have been tested with the following ZIP capable utilities
to check the backward compatibility (non exhaustive list!):

Compatible readers:

* Info-ZIP [unzip](https://infozip.sourceforge.net/UnZip.html) command line utility.

* [libzip](https://libzip.org/): C library for reading, creating, and modifying zip archives

* ``7zip`` command line utility or graphical interface.

* ``WinZip`` graphical interface.

* ``WinRAR`` graphical interface.

* Windows Explorer default ZIP reader.

* MacOSX default ZIP extractor.

* MacOSX ``zipinfo`` command line utility

* [ark](https://apps.kde.org/en/ark/) KDE (Linux/Unix typically) graphical
  interface

* [file-roller](https://gitlab.gnome.org/GNOME/file-roller) GNOME
  (Linux/Unix typically) graphical interface

* Java's [java.util.zip.ZipFile](https://docs.oracle.com/javase/8/docs/api/java/util/zip/ZipFile.html) class.

* Java's [java.util.zip.ZipInputStream](https://docs.oracle.com/javase/8/docs/api/java/util/zip/ZipInputStream.html) class,
  with the caveat that it sees the hidden index files (being a streaming reader,
  it only takes into account local file records)

* Python [zipfile](https://docs.python.org/3/library/zipfile.html) module

* GDAL [/vsizip/](https://gdal.org/user/virtual_file_systems.html#vsizip-zip-archives)
  virtual file system, in all existing GDAL versions, can read SOZip-enabled files.
  Versions >= 3.7 will be able to take advantage of the SOZip index (earlier
  verions will ignore it.)


Partially compatible readers:

* [zipdetails](https://perldoc.perl.org/zipdetails) requires the use of its
  [main](https://github.com/pmqs/zipdetails) branch, or a version greater than
  2.108. Currently released versions (2.108 or earlier) will error out on SOZip
  files, rejecting them because the Local header record of a .sozip.idx file has
  no matching Central header record.


Compatible writers:

* Info-ZIP [zip](https://infozip.sourceforge.net/Zip.html) command line utility,
  used to create / edit zip files, can  append new files to a SOZip-enabled file,
  if using the -g/--grow option, while preserving their existing SOZip-optimization

* Python [zipfile](https://docs.python.org/3/library/zipfile.html) module can
  append new files to a SOZip-enabled file, while preserving their existing
  SOZip-optimization.

* GDAL [sozip](https://github.com/rouault/gdal/blob/sozip/doc/source/programs/sozip.rst)
  command line utility can create SOZip-enabled files, and append new files to
  a SOZip-enabled file, while preserving their existing SOZip-optimization.

* GDAL [/vsizip/](https://gdal.org/user/virtual_file_systems.html#vsizip-zip-archives)
  virtual file system, in all existing GDAL versions, can append new files to a
  SOZip-enabled file, while preserving their existing SOZip-optimization.

However a number of writers, while attempting to "append" to a SOZip-enabled file,
actually create a new file from scratch, and will loose the hidden index.

# Annex E: Pseudo-code for SOZip Deflate stream generation

Licensed under [CC0](https://spdx.org/licenses/CC0-1.0.html) or
[MIT](https://spdx.org/licenses/MIT.html) at the choice of the user (that is feel free
to reuse and adapt without any constaint).

```c++
    // Setup zlib
    z_stream zStream;
    memset(&zStream, 0, sizeof(zStream));

    int ret = deflateInit2(&zStream, Z_DEFAULT_COMPRESSION,
                           Z_DEFLATED, -MAX_WBITS, 8,
                           Z_DEFAULT_STRATEGY);
    // TODO: add error checking

    // Note: for a file whose compressed size does not exceed 4 GB, this could be
    // a vector of uint32_t values.
    std::vector<uint64_t> offsets;
    const uint64_t start_offset = tell_position(compressed_file_handle);

    bool first_block = true;
    uint32_t crc32Val = 0;
    while (true)
    {
        if (!first_block)
        {
            // Track the offset in the compressed stream of the beginning of
            // the new chunk (except for first chunk)
            offsets.push_back(uint64_little_endian(
                tell_position(compressed_file_handle) - start_offset));
        }
        first_block = false;

        // Acquire input data, up to chunk_size bytes
        std::vector<uint8_t> uncompressed_data = read(uncompressed_stream, chunk_size);

        // Prepare output buffer (with security margin for very small chunks,
        // or chunks that compresses poorly)
        std::vector<uint8_t> compressed_data(uncompressed_data.size() +
                                             uncompressed_data.size() / 10 + 16);

        // Compress data
        zStream.avail_in = static_cast<uInt>(uncompressed_data.size());
        zStream.next_in = &uncompressed_data[0];
        zStream.avail_out = static_cast<uInt>(compressed_data.size());
        zStream.next_out = &compressed_data[0];

        if (uncompressed_data.size() < chunk_size)
        {
            ret = deflate(&zStream, Z_FINISH);
            // TODO: add error checking
        }
        else
        {
            ret = deflate(&zStream, Z_SYNC_FLUSH);
            // TODO: add error checking
            ret = deflate(&zStream, Z_FULL_FLUSH);
            // TODO: add error checking
        }

        // Write output buffer
        compressed_data.resize(compressed_data.size() - zStream.avail_out);
        write(compressed_file_handle, compressed_data);

        // Update crc32 of uncompressed data
        crc32Val = crc32(crc32Val, &uncompressed_data[0],
                         static_cast<uInt>(uncompressed_data.size()));

        // Exit loop if no more input data
        if (uncompressed_data.size() < chunk_size)
        {
            break;
        }
    }

    // TODO: store crc32Val in local and central header records

    // Cleanup
    deflateEnd(&zStream);
```

# Annex F: Examples

Examples of SOZip-enabled files can be found in the
[sozip-examples](https://github.com/sozip/sozip-examples) repository.

# Annex G: commented dump of a dummy SOZip file

The following invokation of GDAL's sozip utility generates a dummy
SOZip enabled file that contains a tiny file "foo" with "foo" as content,
and using a chunk size of 2 bytes.

```shell
printf "foo" > foo
sozip --overwrite --enable-sozip=yes --sozip-chunk-size=2 foo.zip foo
```

1. Dump of the local file record of the SOZip compressed "foo" file:

| Offset | Type   | Values                   | Comment                                              |
| ------ | ------ | ------------------------ | ---------------------------------------------------- |
|    0   | uint32 | 0x04034B50               | Local file header signature<br>(for the compressed file) |
|    4   | uint16 | 20                       | version needed to extract |
|    6   | uint16 | 0                        | general purpose bit flag |
|    8   | uint16 | 8                        | compression method (8=deflate) |
|   10   | uint16 | 747                      | last mod file time |
|   12   | uint16 | 21908                    | last mod file date |
|   14   | uint32 | 0x8C736521               | CRC32 |
|   18   | uint32 | 16                       | compressed size |
|   22   | uint32 | 3                        | uncompressed size |
|   26   | uint16 | 3                        | filename length |
|   28   | uint16 | 0                        | extra field length |
|   30   | byte[] | 'f', 'o', 'o'            | filename |
|   33   | byte[] | 0x4A 0xCB 0x07           | deflate block for first chunk (corresponding to 'f', 'o') |
|   36   | byte[] | 0x00 0x00 0x00 0xFF 0xFF | uncompressed block of size 0 encoding a Z_SYNC_FLUSH |
|   41   | byte[] | 0x00 0x00 0x00 0xFF 0xFF | uncompressed block of size 0 encoding a Z_FULL_FLUSH |
|   46   | byte[] | 0xCB 0x07 0x00           | deflate block for second chunk (corresponding to 'o') |


2. Dump of the local file record of the uncompressed ".foo.sozip.idx" index file:

| Offset | Type   | Values                   | Comment                                              |
| ------ | ------ | ------------------------ | ---------------------------------------------------- |
|   49   | uint32 | 0x04034B50               | Local file header signature<br>(for the index file) |
|   53   | uint16 | 20                       | version needed to extract |
|   55   | uint16 | 0                        | general purpose bit flag |
|   57   | uint16 | 0                        | compression method (0=uncompressed) |
|   59   | uint16 | 747                      | last mod file time |
|   61   | uint16 | 21908                    | last mod file date |
|   63   | uint32 | 0xC4707ACE               | CRC32 |
|   67   | uint32 | 36                       | compressed size |
|   71   | uint32 | 36                       | uncompressed size |
|   75   | uint16 | 14                       | filename length |
|   77   | uint16 | 0                        | extra field length |
|   79   | byte[] | '.', 'f', 'o', 'o',<br>'.', 's', 'o', 'z', 'i', 'p',<br>'.', 'i', 'd', 'x' | filename |
|   93   | uint32 | 1                        | SOZip version |
|   97   | uint32 | 0                        | Bytes to skip |
|  101   | uint32 | 2                        | chunk_size |
|  105   | uint32 | 4                        | offset_size |
|  109   | uint64 | 3                        | uncompress_size |
|  117   | uint64 | 16                       | compress_size |
|  125   | uint32 | 13                       | offset of the second compressed chunk<br>(absolute offset is 33 + 13 = 46) |


3. Dump of the central header record describing the "foo" file:

| Offset | Type   | Values                   | Comment                                              |
| ------ | ------ | ------------------------ | ---------------------------------------------------- |
|  129   | uint32 | 0x02014B50               | Central file header signature<br>(compressed file) |
|  133   | uint16 | 0                        | version made by |
|  135   | uint16 | 20                       | version needed to extract |
|  137   | uint16 | 0                        | general purpose bit flag |
|  139   | uint16 | 8                        | compression method (8=deflate) |
|  141   | uint16 | 747                      | last mod file time |
|  143   | uint16 | 21908                    | last mod file date |
|  145   | uint32 | 0x8C736521               | CRC32 |
|  149   | uint32 | 16                       | compressed size |
|  153   | uint32 | 3                        | uncompressed size |
|  157   | uint16 | 3                        | file name length |
|  159   | uint16 | 0                        | extra field length |
|  161   | uint16 | 0                        | file comment length |
|  163   | uint16 | 0                        | disk number start |
|  165   | uint16 | 0                        | internal file attributes |
|  167   | uint32 | 0                        | external file attributes |
|  171   | uint32 | 0                        | relative offset of local header (0 here because this<br>is the first file, and there's no leading content) |
|  175   | byte[] |'f', 'o', 'o'             | filename |


4. Dump of the end of central directory record:

| Offset | Type   | Values                   | Comment                                              |
| ------ | ------ | ------------------------ | ---------------------------------------------------- |
|  178   | uint32 | 0x06054B50               | end of central dir signature |
|  182   | uint16 | 0                        | number of this disk |
|  184   | uint16 | 0                        | number of the disk with the start of the<br>central directory |
|  186   | uint16 | 1                        | total number of entries in the central<br>directory on this disk |
|  188   | uint16 | 1                        | total number of entries in the central<br>directory |
|  190   | uint32 | 49                       | size of the central directory (= 178 - 129) |
|  194   | uint32 | 129                      | offset of start of central directory with<br>respect to the starting disk number |
|  198   | uint16 | 0                        | .ZIP file comment length |


Output of "zipdetails foo.zip" (using main branch of https://github.com/pmqs/zipdetails):

```
0000 LOCAL HEADER #1       04034B50
0004 Extract Zip Spec      14 '2.0'
0005 Extract OS            00 'MS-DOS'
0006 General Purpose Flag  0000
     [Bits 1-2]            0 'Normal Compression'
0008 Compression Method    0008 'Deflated'
000A Last Mod Time         559F56D1 'Sat Dec 31 11:54:34 2022'
000E CRC                   8C736521
0012 Compressed Length     00000010
0016 Uncompressed Length   00000003
001A Filename Length       0003
001C Extra Length          0000
001E Filename              'foo'
0021 PAYLOAD               J...............

0031 LOCAL HEADER #2       04034B50
     Orphan Entry: No
     matching central
     directory
0035 Extract Zip Spec      14 '2.0'
0036 Extract OS            00 'MS-DOS'
0037 General Purpose Flag  0000
0039 Compression Method    0000 'Stored'
003B Last Mod Time         559F56D1 'Sat Dec 31 11:54:34 2022'
003F CRC                   C4707ACE
0043 Compressed Length     00000024
0047 Uncompressed Length   00000024
004B Filename Length       000E
004D Extra Length          0000
004F Filename              '.foo.sozip.idx'

WARNING!
Expected Zip header not found at offset 0x5D
Skipping 0x20 bytes to Central Directory...

0081 CENTRAL HEADER #1     02014B50
0085 Created Zip Spec      00 '0.0'
0086 Created OS            00 'MS-DOS'
0087 Extract Zip Spec      14 '2.0'
0088 Extract OS            00 'MS-DOS'
0089 General Purpose Flag  0000
     [Bits 1-2]            0 'Normal Compression'
008B Compression Method    0008 'Deflated'
008D Last Mod Time         559F56D1 'Sat Dec 31 11:54:34 2022'
0091 CRC                   8C736521
0095 Compressed Length     00000010
0099 Uncompressed Length   00000003
009D Filename Length       0003
009F Extra Length          0000
00A1 Comment Length        0000
00A3 Disk Start            0000
00A5 Int File Attributes   0000
     [Bit 0]               0 'Binary Data'
00A7 Ext File Attributes   00000000
00AB Local Header Offset   00000000
00AF Filename              'foo'

00B2 END CENTRAL HEADER    06054B50
00B6 Number of this disk   0000
00B8 Central Dir Disk no   0000
00BA Entries in this disk  0001
00BC Total Entries         0001
00BE Size of Central Dir   00000031
00C2 Offset to Central Dir 00000081
00C6 Comment Length        0000

WARNINGS

* Expected Zip header not found at offset 0x61, skipped 0x20 bytes

Done
```
