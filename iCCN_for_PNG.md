# iCCN

## iCCN Embedded ICC profile (updated)
The four-byte chunk type field contains the decimal values

`105 67 67 80`

The iCCN chunk contains:

| Profile name |  1-79 bytes (character string)
| Null separator |	1 byte (null character)
| Compression method |	1 byte
| Compressed profile |	n bytes

The profile name may be any convenient name for referring to the profile. It is case-sensitive. Profile names shall be encoded as UTF-8. Leading, trailing, and consecutive spaces are not permitted. The only compression method defined in this International Standard is method 0 (zlib datastream with deflate compression, see 10.3: Other uses of compression). The compression method entry is followed by a compressed datastream of an ICC profile as defined in [ICC] or [ICC-2001]. The ICC profile shall either be an output profile (Device Class = `prtr`) or a monitor profile (Device Class = `mntr`). Decompression of this datastream yields the embedded ICC profile.

If the iCCN chunk is present, the image samples conform to the colour space represented by the embedded ICC profile. The colour space of the ICC profile shall be an RGB (`RGB `) colour space for colour images (PNG colour types 2, 3, and 6), or a greyscale (`GRAY`) colour space for greyscale images (PNG colour types 0 and 4). A PNG encoder that writes the iCCN chunk is encouraged to also write `gAMA` and `cHRM` chunks that approximate the ICC profile, to provide compatibility with applications that do not use the iCCN chunk. When the iCCN chunk is present, PNG decoders that recognize it and are capable of colour management shall ignore any `gAMA`, `cHRM`, and `CiCP` chunks and use the iCCN chunk instead and interpret it according to [ICC] or [ICC-2001] as appropriate. PNG decoders that are used in an environment that is incapable of full-fledged colour management should use the gAMA and cHRM chunks if present.

A PNG datastream shall contain at most one embedded profile, whether specified explicitly with an `iCCP` or `iCCN` chunk or implicitly with an `sRGB` chunk.

## Normative References

[ICC] ISO 15076-1:2010, Image technology colour management – Architecture, profile format and data structure — Part 1: Based on ICC.1:2010

[ICC-2001] Specification ICC.1:2001-04, File Format for Color Profiles

