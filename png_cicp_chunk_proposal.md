# Adding support for HDR imagery to the PNG format
Editors: Chris Blume, Pierre-Anthony Lemieux, Chris Seeger, Leonard Rosenthol

Status: Draft

## Problem to be solved
The gAMA chunk of the Portable Network Graphics (PNG) format (specified in [PNG]) parameterized the transfer function of the image as a power law. As such, it cannot model the Reference PQ or HLG OOTFs specified in [BT2100-1], which are commonly used for HDR images.

An existing W3C group note, [BT2100-in-PNG]  specifies an approach which is limited: it supports signaling of only the ITU BT.2100 PQ EOTF and uses magic values in the iCCP chunk to signal color spaces.

## Basic requirements
* does not break current implementations
* extensible signaling of color space based on H.273
* does not require the presence of iCCP chunk and embedded ICC profiles

## Non Requirements
* bit-identical serialization of the H.273 color space parameters as used by other raster image formats (eg. JPEG, AVIF)

## Strawman approach
[H.273](https://www.itu.int/rec/T-REC-H.273/en) specifies a controlled vocabulary for the parameterization of color space information.

Define a `cICP` chunk that contains the 3 bytes necessary to carry the H.273 color space parameters:

* **COLPRIMS**, 1 byte, One of the ColourPrimaries enumerated values specified in Rec. ITU-T H.273 | [ISO/IEC 23091-2]
* **TRANSFC**, 1 byte, One of the TransferCharacteristics enumerated values specified in Rec. ITU-T H.273 | [ISO/IEC 23091-2]
* **VIDFRNG**, 1 byte, Value of the VideoFullRangeFlag specified in Rec. ITU-T H.273 | [ISO/IEC 23091-2]

NOTE: While these are inspired from recent JPEG standards (eg. JPEG-XL) that incorporate these color space parameters, this specification is only using 1 byte per value as defined in H.273 (vs. 2 bytes in JPEG-XL).

[ed.: The MATOEFFS parameter is not included because PNG does not support YCbCr]

NOTE: [ITU-T Series H Supplement 19](https://www.itu.int/rec/T-REC-H.Sup19-201910-I) summarize combinations of H.273 parameters corresponding to common baseband linear broadcasts and file-based Video-on-Demand(VOD) services.

The `cICP` chunk SHALL come before `IDAT` chunk.  

A PNG MAY contain both a `cICP` chunk and an `iCCP` chunk. If a PNG decoder detects the presence of both a `cICP` and a `iCCP` chunk, the behavior is undefined.

When the `cICP` chunk is present, a PNG decoder SHALL ignore the following chunks:
- `gAMA`
- `cHRM` 
- `sRGB` 

## A. References

[ITU-T Series H Supplement 19](https://www.itu.int/rec/T-REC-H.Sup19-201910-I). Series H: Audiovisual and multimedia systems - Usage of video signal type code points
[BT2100-1]
[Recommendation ITU-R BT.2100-1](https://www.itu.int/rec/R-REC-BT.2100), Image parameter values for high dynamic range television for use in production and international programme exchange

[ITU-T H.273]
[Technical Document ITU-T H.273](https://www.itu.int/rec/T-REC-H.273/en), Color Independent Coding Points for Images

[PNG]
[Portable Network Graphics (PNG) Specification (Second Edition)](https://www.w3.org/TR/PNG/). Tom Lane. W3C. 10 November 2003. W3C Recommendation. URL: https://www.w3.org/TR/PNG

[BT2100-in-PNG]
[Using the ITU BT.2100 PQ EOTF with the PNG format](https://www.w3.org/TR/png-hdr-pq/)

[ISO/IEC 23091-2]
[ISO/IEC 23091-2](https://www.iso.org/standard/81546.html)
