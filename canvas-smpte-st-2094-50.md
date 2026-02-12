# SMPTE ST 2094-50 Tone mapping for canvas elements

This proposal suggests an interface for adding support for [SMPTE ST 2094-50](https://github.com/SMPTE/st2094-50) tone mapping to `CanvasToneMapping`.
The SMPTE ST 2094-50 standard is currently in public review.
The most recent public draft can be downloaded at the above link, under the "Public Committee Draft (PCD) Notice" section.

This fits into the [HDR on the web, the big picture](hdr-big-picture.md) proposal.
It is the "magenta sun number 5" and "magenta sun number 6" problem in the proposal diagram.

A prerequisite is the `CanvasToneMapping` API is proposed in [this explainer](canvas-tone-map.md).

## Proposal

Add the following dictionaries for the SMPTE ST 2094-50 color volume transform metadata group.

```idl
  dictionary SmpteSt2094App5ColorVolumeTransform {
    required double hdrReferenceWhite;
    optional SmpteSt2094App5HeadroomAdaptiveToneMap headroomAdaptiveToneMap;
  }
  dictionary SmpteSt2094App5HeadroomAdaptiveToneMap {
    required double baselineHdrHeadroom;
    required double gainApplicationColorSpacePrimaries[8];
    required SmpteSt2094App5HeadroomAlternateImage alternateImages[];
  }
  dictionary SmpteSt2094App5HeadroomAlternateImage {
    required double hdrHeadroom;
    SmpteSt2094App5HeadroomColorGainFunction colorGainFunction;
  }
  dictionary SmpteSt2094App5HeadroomColorGainfunction {
    SmpteSt2094App5HeadroomComponentMixingFunction componentMix;
    SmpteSt2094App5HeadroomGainCurve gainCurve;
  }
  SmpteSt2094App5HeadroomComponentMixingFunction {
    required double red = 0;
    required double green = 0;
    required double blue = 0;
    required double max = 0;
    required double min = 0;
    required double component = 0;
  }
  SmpteSt2094App5HeadroomGainCurve {
    SmpteSt2094App5HeadroomGainCurveControlPoint controlPoints[];
  }
  SmpteSt2094App5HeadroomGainCurveControlPoint {
    required double x;
    required double y;
    required double m;
  }
```

To the `CanvasToneMappingMode` enum, add an entry to indicate to use SMPTE ST 2094-50.

```idl
  enum CanvasToneMappingMode {
      "smpte-st-2094-application-5",
  }
```

To the `CanvasToneMapping` dictionary, add an entry for the parameters.

```
  enum CanvasToneMapping {
    optional SmpteSt2094App5ColorVolumeTransform smpteSt2094App5ColorVolumeTransform;
  }
```

Add a corresponding entry to the [WebCodecs video frame metadata registry](https://www.w3.org/TR/webcodecs-video-frame-metadata-registry/).

## Future directions

This mechanism may be used as the basis for other tone mapping mechanisms, as described in the [canvas tone map](canvas-tone-map#future-directions) explainer.
