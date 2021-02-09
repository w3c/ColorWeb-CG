# Adding support for HDR imagery to the PNG format
Authors (alphabetical): Chris Blume, Pierre-Anthony Lemieux, Chris Seeger

Status: Draft

## Problem to be solved
The gAMA chunk of the Portable Network Graphics (PNG) format (specified in [PNG]) parameterized the transfer function of the image as a power law. As such, it cannot model the Reference PQ or HLG OOTFs specified in [BT2100-1], which are commonly used for HDR images.

This specification uses the existing iCCP chunk to unambiguously signal the color system of an image that uses the Reference PQ EOTF specified in [BT2100-1]. It also allows graceful processing by decoders that do not conform to this specification by recommending fallback values for the gAMA chunk, cHRM chunk, and embedded ICC profile.

An existing W3C group note (https://www.w3.org/TR/png-hdr-pq/) specifies an approach which is limited: it supports signaling of only the ITU BT.2100 PQ EOTF and uses magic values in the iCCP chunk to signal color spaces.

[H.273](https://www.itu.int/rec/T-REC-H.273/en) specifies a controlled vocabulary for the parameterization of
color space information:
* Color Primaries
* Transfer Function
* Matrix Coefficients (only required if essence is stored as Y’CbCr)

Use [ITU-T Series H Supplement 19](https://www.itu.int/rec/T-REC-H.Sup19-201910-I) “recommendations” to signal the controlled vocabulary for image formats commonly used for baseband linear broadcasts or file-based Video-on-Demand(VOD) services.
* Tables within this document summarize values most commonly used in broadcast for 47 different standards documents including H.273


## Basic requirements
* does not break current implementations
* extensible signaling of color space based on H.273
* does not require the presence of iCCP chunk and embedded ICC profiles
* cICP chunk comes before IDAT chunk

## Strawman approach
Define a cICP chunk that contains the 7 bytes necessary to carry the
H.273 color space parameters:

* COLPRIMS, 2 bytes, One of the ColourPrimaries enumerated values specified in Rec. ITU-T H.273 | ISO/IEC 23001-8
* TRANSFC, 2 bytes, One of the TransferCharacteristics enumerated values specified in Rec. ITU-T H.273 | ISO/IEC 23001-8
* MATCOEFFS, 2 bytes, One of the MatrixCoefficients enumerated values specified in Rec. ITU-T H.273 | ISO/IEC 23001-8
* VIDFRNG, 1 byte, Value of the VideoFullRangeFlag specified in Rec. ITU-T H.273 | ISO/IEC 23001-8

[ed.: these are inspired from recent JPEG standards that incorporate
H.273 color space parameters]

## A. References
### A.1 Normative references
[BT2100-1]
[Recommendation ITU-R BT.2100-1](https://www.itu.int/rec/R-REC-BT.2100), Image parameter values for high dynamic range television for use in production and international programme exchange

[ITU-T H.273]
[Technical Document ITU-T H.273](https://www.itu.int/rec/T-REC-H.273/en), Color Independent Coding Points for Images

[PNG]
[Portable Network Graphics (PNG) Specification (Second Edition)](https://www.w3.org/TR/PNG/). Tom Lane. W3C. 10 November 2003. W3C Recommendation. URL: https://www.w3.org/TR/PNG
