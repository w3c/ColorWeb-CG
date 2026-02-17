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
    SmpteSt2094App5HeadroomAdaptiveToneMap headroomAdaptiveToneMap;
  };
  dictionary SmpteSt2094App5HeadroomAdaptiveToneMap {
    required double baselineHdrHeadroom;
    required sequence<double> gainApplicationColorSpacePrimaries; // must have exactly 8
    required sequence<SmpteSt2094App5HeadroomAlternateImage> alternateImages;
  };
  dictionary SmpteSt2094App5HeadroomAlternateImage {
    required double hdrHeadroom;
    SmpteSt2094App5HeadroomColorGainFunction colorGainFunction;
  };
  dictionary SmpteSt2094App5HeadroomColorGainfunction {
    SmpteSt2094App5HeadroomComponentMixingFunction componentMix = {};
    SmpteSt2094App5HeadroomGainCurve gainCurve;
  };
  dictionary SmpteSt2094App5HeadroomComponentMixingFunction {
    double red = 0;
    double green = 0;
    double blue = 0;
    double max = 0;
    double min = 0;
    double component = 0;
  };
  dictionary SmpteSt2094App5HeadroomGainCurve {
    required sequence<SmpteSt2094App5HeadroomGainCurveControlPoint> controlPoints; // must have at least 1
  };
  dictionary SmpteSt2094App5HeadroomGainCurveControlPoint {
    required double x;
    required double y;
    required double m;
  };
```

To the `CanvasToneMappingMode` enum, add an entry to indicate to use SMPTE ST 2094-50.

```idl
  enum CanvasToneMappingMode {
      // ...
      "smpte-st-2094-application-5",
  };
```

To the `CanvasToneMapping` dictionary, add an entry for the parameters.

```
  partial dictionary CanvasToneMapping {
    SmpteSt2094App5ColorVolumeTransform smpteSt2094App5ColorVolumeTransform;
  };
```

Add a corresponding entry to the [WebCodecs video frame metadata registry](https://www.w3.org/TR/webcodecs-video-frame-metadata-registry/).

## Future directions

This mechanism may be used as the basis for other tone mapping mechanisms, as described in the [canvas tone map](canvas-tone-map#future-directions) explainer.
