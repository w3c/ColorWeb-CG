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

### cICP chunk

[H.273](https://www.itu.int/rec/T-REC-H.273/en) specifies a controlled vocabulary for the parameterization of color space information.

Define a `cICP` chunk that contains the 3 bytes necessary to carry the H.273 color space parameters:

* **COLPRIMS**, 1 byte, One of the ColourPrimaries enumerated values specified in Rec. ITU-T H.273 | [ISO/IEC 23091-2]
* **TRANSFC**, 1 byte, One of the TransferCharacteristics enumerated values specified in Rec. ITU-T H.273 | [ISO/IEC 23091-2]
* **MATCOEFFS**, 1 byte, One of the MatrixCoefficients enumerated values specified in Rec. ITU-T H.273 | [ISO/IEC 23091-2]
* **VIDFRNG**, 1 byte, Value of the VideoFullRangeFlag specified in Rec. ITU-T H.273 | [ISO/IEC 23091-2]

NOTE: While these are inspired from recent JPEG standards (eg. JPEG-XL) that incorporate these color space parameters, this specification is only using 1 byte per value as defined in H.273 (vs. 2 bytes in JPEG-XL).

NOTE: [ITU-T Series H Supplement 19](https://www.itu.int/rec/T-REC-H.Sup19-201910-I) summarize combinations of H.273 parameters corresponding to common baseband linear broadcasts and file-based Video-on-Demand(VOD) services.

The `cICP` chunk SHALL come before `IDAT` chunk.  

A PNG MAY contain both a `cICP` chunk and an `iCCP` chunk. If a PNG decoder detects the presence of both a `cICP` and a `iCCP` chunk, the behavior is undefined.

When the `cICP` chunk is present, a PNG decoder SHALL ignore the following chunks:
- `gAMA`
- `cHRM` 
- `sRGB` 

### iCCN chunk Embedded ICC profile (updated)

#### Structure

The four-byte chunk type field contains the decimal values

`105 67 67 80`

The `iCCN` chunk contains:

| Profile name |  1-79 bytes (character string)
| Null separator |	1 byte (null character)
| Compression method |	1 byte
| Compressed profile |	n bytes

The profile name may be any convenient name for referring to the profile. It is case-sensitive. Profile names shall be encoded as UTF-8. Leading, trailing, and consecutive spaces are not permitted. The profile name shall not contain a zero byte (null character). 

The only compression method defined in this International Standard is method 0 (zlib datastream with deflate compression, see 10.3: Other uses of compression). The compression method entry is followed by a compressed datastream of an ICC profile as defined in [ICC] or [ICC-2010]. The ICC profile shall either be an output profile (Device Class = `prtr`) or a monitor profile (Device Class = `mntr`). Decompression of this datastream yields the embedded ICC profile.

NOTE: This is exactly the same as `iCCP` except the profile name is UTF-8 instead of Latin-1. Analogous to `tEXt` vs. `iTXt`

#### Encoder

If the `iCCN` chunk is present, the image samples conform to the colour space represented by the embedded ICC profile. The colour space of the ICC profile shall be an RGB (`RGB `) colour space for colour images (PNG colour types 2, 3, and 6), or a greyscale (`GRAY`) colour space for greyscale images (PNG colour types 0 and 4). A PNG encoder that writes the `iCCN` chunk is encouraged to also write `gAMA` and `cHRM` chunks that approximate the ICC profile, to provide compatibility with applications that do not use the iCCN chunk. 

### Decoder

If the image contains a `cICP` chunk and will be rendered to a display or surface that supports `cICP`, then the PNG decoder shall ignore any `gAMA`, `cHRM`, and `iCCN` chunks and use the `cICP` chunk instead. Otherwise, when a `iCCN` chunk is present, PNG decoders that recognize it and are capable of colour management shall ignore any `gAMA`, `cHRM`, and `cICP` chunks and use the `iCCN` chunk instead and interpret it according to [ICC] or [ICC-2010] as appropriate. PNG decoders that are used in an environment that is incapable of full-fledged colour management shall use the `gAMA` and `cHRM` chunks if present.

#### Codestream

A PNG datastream shall contain at most one embedded profile, whether specified explicitly with an `iCCP` or `iCCN` chunk or implicitly with an `sRGB` chunk.

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
