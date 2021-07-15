# Adding support for HDR imagery to the PNG format
Editors: Chris Blume, Pierre-Anthony Lemieux, Chris Seeger, Leonard Rosenthol

Status: Draft

## Problem to be solved
The gAMA chunk of the Portable Network Graphics (PNG) format (specified in [PNG]) parameterized the transfer function of the image as a power law. As such, it cannot model the Reference PQ or HLG OOTFs specified in [BT2100], which are commonly used for HDR images.

An existing W3C group note, [BT2100-in-PNG]  specifies an approach which is limited: it supports signaling of only the ITU BT.2100 PQ EOTF and uses magic values in the iCCP chunk to signal color spaces.

## Basic requirements
* does not break current implementations
* extensible signaling of color space based on H.273
* does not require the presence of iCCP chunk and embedded ICC profiles

## Non Requirements
* bit-identical serialization of the H.273 color space parameters as used by other raster image formats (eg. JPEG, AVIF)

## Strawman approach

Two new PNG chunks are proposed, the `cICP` chunk and the `iCCN` chunk.

The `cICP` chunk acts as a color space label (much like the existing `sRGB` chunk), specifying the color space to which the pixels within the PNG file conforms. The `iCCN` chunk (much like the existing `iCCP`) contains an embedded color profile.

A PNG may contain multiple chunks with color space information. A PNG viewer should use the highest priority color space chunk that it can honor, ignoring the others. The priorities are from highest to lowest:

* `cICP`
* `iCCN`
* `iCCP`
* `gAMA`
* `cHRM`
* `sRGB`

### cICP chunk

This chunk SHALL come before the `IDAT` chunk.

[H.273](https://www.itu.int/rec/T-REC-H.273/en) specifies a controlled vocabulary for the parameterization of color space information.

Define a `cICP` chunk that contains the 4 bytes necessary to carry the H.273 color space parameters:

* **COLPRIMS**, 1 byte, One of the ColourPrimaries enumerated values specified in Rec. ITU-T H.273 | [ISO/IEC 23091-2]
* **TRANSFC**, 1 byte, One of the TransferCharacteristics enumerated values specified in Rec. ITU-T H.273 | [ISO/IEC 23091-2]
* **MATCOEFFS**, 1 byte, One of the MatrixCoefficients enumerated values specified in Rec. ITU-T H.273 | [ISO/IEC 23091-2]
* **VIDFRNG**, 1 byte, Value of the VideoFullRangeFlag specified in Rec. ITU-T H.273 | [ISO/IEC 23091-2]

NOTE: While these are inspired from recent JPEG standards (eg. JPEG-XL) that incorporate these color space parameters, this specification follows H.273.

NOTE: [ITU-T Series H Supplement 19](https://www.itu.int/rec/T-REC-H.Sup19-201910-I) summarize combinations of H.273 parameters corresponding to common baseband linear broadcasts and file-based Video-on-Demand(VOD) services.

### iCCN chunk Embedded ICC profile (updated)

This chunk SHALL come before the `IDAT` chunk.

#### Structure

The `iCCN` chunk contains:

|Name               |Size                         |
|-------------------|-----------------------------|
|Profile name       |1-79 bytes (character string)|
|Null separator     |1 byte (null character)      |
|Compression method |1 byte                       |
|Compressed profile |n bytes                      |

The profile name may be any convenient name for referring to the profile. It is case-sensitive. Profile names shall be encoded as UTF-8. Leading, trailing, and consecutive spaces are not permitted. The profile name shall not contain a zero byte (null character). 

The compression method shall be method 0 (zlib datastream with deflate compression). The compression method entry is followed by a compressed datastream of an ICC profile as defined in [ICC] or [ICC-2010].

The ICC profile shall either be an output profile (Device Class = `prtr`) or a monitor profile (Device Class = `mntr`). Decompression of this datastream yields the embedded ICC profile.

NOTE: This is exactly the same as `iCCP` except:

* `iCCP` is ICCv2 (although many decoders treat it as ICCv4) while `iCCN` is explicitly ICCv4
* the profile name is UTF-8 instead of Latin-1. Analogous to `tEXt` vs. `iTXt`

#### Encoder

If the `iCCN` chunk is present, the image samples conform to the colour space represented by the embedded ICC profile. The colour space of the ICC profile shall be an RGB (`RGB `) colour space for colour images (PNG colour types 2, 3, and 6), or a greyscale (`GRAY`) colour space for greyscale images (PNG colour types 0 and 4). A PNG encoder that writes the `iCCN` chunk is encouraged to also write `gAMA` and `cHRM` chunks that approximate the ICC profile, to provide compatibility with applications that do not use the iCCN chunk. 

### Decoder

The `iCCN` chunk should be interpreted according to [ICC] or [ICC-2010] as appropriate. 

## References

[ICC] ISO 15076-1:2010, Image technology colour management – Architecture, profile format and data structure — Part 1: Based on ICC.1:2010

[ICC-2010] Specification ICC.1:2010-12 (Profile version 4.3.0.0) Image technology colour management - Architecture, profile format, and data structure

[ITU-T Series H Supplement 19](https://www.itu.int/rec/T-REC-H.Sup19-201910-I). Series H: Audiovisual and multimedia systems - Usage of video signal type code points

[BT2100]
[Recommendation ITU-R BT.2100](https://www.itu.int/rec/R-REC-BT.2100), Image parameter values for high dynamic range television for use in production and international programme exchange

[ITU-T H.273]
[Technical Document ITU-T H.273](https://www.itu.int/rec/T-REC-H.273/en), Color Independent Coding Points for Images

[PNG]
[Portable Network Graphics (PNG) Specification (Second Edition)](https://www.w3.org/TR/PNG/). Tom Lane. W3C. 10 November 2003. W3C Recommendation. URL: https://www.w3.org/TR/PNG

[BT2100-in-PNG]
[Using the ITU BT.2100 PQ EOTF with the PNG format](https://www.w3.org/TR/png-hdr-pq/)

[ISO/IEC 23091-2]
[ISO/IEC 23091-2](https://www.iso.org/standard/81546.html)
