# Adding support High Dynamic Range (HDR) imagery to HTML Canvas

## Introduction

Today [HTML
Canvas](https://html.spec.whatwg.org/multipage/canvas.html#imagedata) supports
only 8 bit per color channel and two `PredefinedColorSpace` color spaces (`srgb`
and `display-p3`).

This is insufficient for High-Dynamic Range (HDR) imagery, which is in
widespread use today:

* As detailed, for example, at [Ultra HD Blu-ray Format Video Characteristics
](https://ieeexplore.ieee.org/document/7514362), 8-bit quantization (bit depth)
results in contouring and banding, even for traditional standard dynamic range
(SDR) imagery, like sRGB, which covers a typical luminance range between 0 and
100 cd/m<sup>2</sup>. These quantization artifacts become unacceptable with
High-Dynamic Range (HDR) imagery, supports luminance ranges between 0 and up to
10,000 cd/m<sup>2</sup>.

* As specified at [Rec. ITU-R BT.2100](https://www.itu.int/rec/R-REC-BT.2100),
two color spaces tailored for HDR imagery have developed: BT.2100 PQ and BT.2100
HLG.

* To render HDR imagery, it is useful to have information on the luminance range
and color gamut that (a) are supported by the display and (b) were used when
authoring the image.

Accordingly, the following API modifications are needed to manipulate HDR images
in HTML Canvas:

1. add the ability to query the luminance range and color gamut of the display
2. add BT.2100 PQ and BT.2100 HLG color spaces to `PredefinedColorSpace`
3. add higher bit depth capabilities to `CanvasRenderingContext2DSettings`
4. add higher bit depth capabilities to `ImageDataSettings`
5. add luminance and color gamut information to `ImageDataSettings` and
   `CanvasRenderingContext2DSettings`

## Scope

We propose to extend the Web Platform to allow the HTML Canvas API to manipulate
High Dynamic Range (HDR) images.

## Add the ability to query the luminance range and color gamut of the display

### Querying screen HDR parameters

Add a new attribute to `ScreenAdvanced` to indicate the HDR headroom currently
available on the screen.

```idl
  partial interface ScreenAdvanced {
    // The maximum luminance that the screen is capable of displaying across
    // the full area of the screen, as a multiple of the luminance of SDR white.
    // This will have a value of 1.0 for screens that are not HDR capable.
    readonly attribute double highDynamicRangeHeadroom;
  }
```

### Querying screen wide color gamut parameters

Add new attributes to `ScreenDetailed` to indicate the gamut and white point of
the screen.

```idl
  partial interface ScreenDetailed {
    // The color volume that the screen is capable of displaying.
    readonly attribute ColorVolume colorVolume;
  }
```

```idl
  dictionary ColorVolume {
    // The color primaries and white point of a color volume, in CIE 1931 xy
    // coordinates.
    required double redPrimaryX;
    required double redPrimaryY;
    required double greenPrimaryX;
    required double greenPrimaryY;
    required double bluePrimaryX;
    required double bluePrimaryY;
    required double whitePointX;
    required double whitePointY;
  }
```

## Add color spaces intended for use with HDR imagery

### General

Extend `PredefinedColorSpace` to include the following color spaces.

```idl
  partial enum PredefinedColorSpace {
    'rec2100-hlg',
    'rec2100-pq',
    'linear-srgb'
  }
```

As illustrated below, the tone mapping of `rec2100-pq` and `rec2100-hlg` images,
i.e. the rendering of an image with a given dynamic range onto a display with
different dynamic range, is performed by the platform. This is akin to the
scenario where the `src` of an `img` element is a PQ or HLG image.

![Tone mapping scenarios](./tone-mapping-scenarios.png)

Conversions to and from `rec2100-pq` and `rec2100-hlg` are detailed in Annex A
below.

Extending `PredefinedColorSpace` automatically extends
`CanvasRenderingContext2DSettings` and `ImageDataSettings`.

### rec2100-hlg

The component signals are mapped to red, green and blue tristimulus values
according to the Hybrid Log-Gamma (HLG) system specified in Rec. ITU-R BT.2100.

### rec2100-pq

The component signals are mapped to red, green and blue tristimulus values
according to the Perceptual Quantizer (PQ) system system specified in Rec. ITU-R
BT.2100.

### linear-srgb

`linear-srgb` is specified in CSS Color 4 and is useful for physics-based
rendering of HDR scenes.

### Extend `CanvasRenderingContext2DSettings` to support higher bit depths

Add to `CanvasRenderingContext2DSettings` a `CanvasColorType` member that
specifies the representation of each pixel of the _output bitmap_ of a
`CanvasRenderingContext2D` and `OffscreenCanvasRenderingContext2D`.

```idl
  partial dictionary CanvasRenderingContext2DSettings {
    CanvasColorType colorType = "unorm8";
  };
```

```idl
  enum CanvasColorType {
    "unorm8",
    "float16",
  };
```

`colorType = "unorm8"` corresponds to HTML Canvas as it exists today.

### Extend `ImageDataSettings` to support higher bit depths

Add to `ImageDataSettings` a `ImageDataColorType` member that specifies the type
of the `data` member of `ImageData`.

```idl
  partial dictionary ImageDataSettings {
    ImageDataColorType colorType = "Uint8ClampedArray";
  };
```

```idl
  enum ImageDataColorType {
    "Uint8ClampedArray",
    "Float16Array",
    "Float32Array"
    // and potentially others
  };
```

## Add luminance and color gamut information to `ImageDataSettings` and `CanvasRenderingContext2DSettings`

Add a new CanvasColorMetadata dictionary:

  dictionary CanvasColorMetadata {
    CanvasHighDynamicRangeMode mode = 'default';
    CanvasSmpteSt2086Metadata smpteSt2086Metadata;
  }

  enum CanvasHighDynamicRangeMode {
    // The default behavior.
    // HDR is enabled for 'rec2100-hlg' and 'rec2100-pq' color spaces only.
    'default',
    // HDR is enabled for all color spaces
    'extended',
  }

  // SMPTE ST 2086 color volume metadata.
  dictionary CanvasSmpteSt2086Metadata {
    ColorVolume colorVolume;
    required float minimumLuminanceNits;
    required float maximumLuminanceNits;
  }

Add a mechanism for specifying this on `CanvasRenderingContext2D` and
`OffscreenCanvasRenderingContext2D`.

  partial interface CanvasRenderingContext2D/OffscreenCanvasRenderingContext2D {
    attribute CanvasColorMetadata colorMetadata;
  }

The semantics of `CanvasHighDynamicRangeMode` are as follows:

* Define a *screen's native linear color space* to be the color space with color
primaries and white point set to the screen's `ScreenDetailed`'s `colorVolume`
and the identity as its transfer function.

* A color is said to be within a *screen's default range* if, when that color is
converted to the *screen's native linear color space*, all of its color
components are within the `[0, 1]` interval. When displaying content produced by
a rendering context with `CanvasHighDynamicRangeMode` set to `'default'`, all
colors that are within the *screen's default range* must not be gamut-mapped;
other colors may be clipped or subject to screen-specific behavior.

* A color is said to be within a screen's *extended range* if, when that color
is converted to the *screen's native linear color space*, all of its color
components are within the `[0, highDynamicRangeHeadroom]` interval. When
displaying content produced by a rendering context with
`CanvasHighDynamicRangeMode` set to `'extended'`, all colors that are within the
*screen's extended range* must not be gamut-mapped; other colors may be clipped
or subject to screen-specific behavior.

## Annex A: Color space conversions

### Background

In general, application should avoid conversions between color spaces and
maintain imagery in its original color space: conversions between color spaces
are not necessarily reversible and do not necessarily result in the same image
appearance. In particular, conversion of an HDR image to an SDR will result in a
signification loss of information and an SDR image that is different from the
SDR image that would have been mastered from the same source material. From that
perspective, converting from HDR to SDR imagery is similar to converting RGBA
images to 16-color pallette images.

Nevertheless, the HTML specification allows color space conversion in several
scenarios, e.g., when [drawing images to a
canvas](https://html.spec.whatwg.org/multipage/canvas.html#colour-spaces-and-colour-correction),
[retrieving image data from a
canvas](https://html.spec.whatwg.org/multipage/canvas.html#dom-context-2d-getimagedata),
among others being added). The conversions between predefined SDR color spaces
are defined at <https://www.w3.org/TR/css-color-4/>, and this proposal similarly
defines conversions for HDR color spaces.

The following illustrates the conversions that are explicitly specified:

![Color space conversions](./color-space-conversions.png)

These conversions fall into two broad categories:

* conversion between HDR color spaces
* conversion between an HDR and an SDR color space (tone mapping)

### Between HDR color spaces

The conversion between `rec2100-pq` and `rec2100-hlg` is specified at [Report
ITU-R BT.2408-5, Clause 6](https://www.itu.int/pub/R-REP-BT.2408),

### Between SDR and HDR color spaces

#### General

Conversions to and from `srgb` are provided for `rec2100-pq` and `rec2100-hlg` color spaces.

#### `rec2100-pq` to `srgb`

Tone mapping from `rec2100-pq` to `srgb` is specified at [SMPTE ST 2094-10,
Annex B](https://ieeexplore.ieee.org/document/7513370).

#### `rec2100-hlg` to `srgb`

_Input:_ Full-range non-linear floating-point `rec2100-hlg` pixel with black at
0.0 and diffuse white at 0.75. Values may exist outside the range 0.0 to 1.0.

_Output:_ Full-range non-linear floating-point `srgb` pixel with black at 0.0
and diffuse white at 1.0. Values may exist outside the range 0.0 to 1.0.

_Process:_
  1. Pseudo-linearize the HLG signal exploiting its backwards compatibility with
     SDR consumer displays
  2. Convert from ITU BT.2100 color space to sRGB color space
  3. Convert back to non-linear using a reciprocal transform

_Note 3_ This transform utilises the backwards compatibility of ITU-R BT.2100
HLG HDR with consumer electronic displays.  Prior to display, the gamut may need
to be limited to the range 0-1.  The simplest method is to clip values but other
gamut reduction techniques may provide better output images.

```javascript

function simpleTransform(value, systemGamma) {
  if (value < 1.0) {
    return -1.0 * Math.pow(-1.0 * value, systemGamma);
  } else {
    return Math.pow(value, systemGamma);
  } 
}

function simpleInverseTransform(value, systemGamma) {
  if (value < 1.0) {
    return -1.0 * Math.pow(-1.0 * value, 1.0 / systemGamma);
  } else {
    return Math.pow(value, 1.0 / systemGamma);
  }
}

function tonemapREC2100HLGtoSRGBdisplay(r, g, b) {
  const systemGamma = 2.2;
  const r1 = simpleTransform(r, systemGamma);
  const g1 = simpleTransform(g, systemGamma);
  const b1 = simpleTransform(b, systemGamma);
  const [r2, g2, b2] = matrixXYZtoSRGB(matrixBT2020toXYZ(r1, g1, b1));
  const r3 = simpleInverseTransform(r2, systemGamma);
  const g3 = simpleInverseTransform(g2, systemGamma);
  const b3 = simpleInverseTransform(b2, systemGamma);
  const [r4, g4, b4] = limitTosRGBGamut(r3, g3, b3);
  return [r4, g4, b4];
}
```

#### `srgb` to `rec2100-hlg`

See [TTML 2, Annex Q.2, steps 1-8](https://www.w3.org/TR/ttml2/#hlg-hdr) with
`tts:luminanceGain = 203/80`.

#### `srgb` to `rec2100-pq`

See [TTML 2, Annex Q.1, steps 1-8](https://www.w3.org/TR/ttml2/#hdr-compositing).

## Annex B: Compositing the HDR `HTMLCanvasElement`

The compositing behavior of an `HTMLCanvasElement` may be specified via the
`CanvasColorMetadata` attribute of its rendering context.

### Default behavior

This section describes the default compositing behavior for canvas element. This
is the behavior that happens if the `CanvasColorMetadata` is not set, or if is
called specifying `'default'` as the mode.

In this mode, the `HTMLCanvasElement` will be composited exactly as an `<img>`
or `<video>` element with a source in the canvas' color space would be
composited.

This means that if the canvas' color space is `'rec2100-hlg'` or `'rec2100-pq'`,
then the canvas will be composited using high dynamic range, where available.

Otherwise, e.g. if the canvas' color space is `'srgb-linear'` or `'srgb'`, the
canvas will not be composited using high dynamic range. Pixel values outside of
the [0, 1] interval will extend the displayed gamut beyond sRGB, but not the
displayed luminance beyond the maximum SDR luminance.

Performing appropriate tone mapping is the responsibility of the browser, the
operating system, and the display device. If the `CanvasColorMetadata` is set to
specify metadata and the canvas' color space is `'rec2100-pq'`, then this
metadata will be interpreted during compositing in the same way that it would be
interpreted if included in a source displayed via an `<img>` or `<video>`
element.

### Extended mode

If the canvas' color space is `'srgb-linear'` or `'srgb'`, then pixels values
outside of the [0, 1] interval will extend the displayed luminance beyond the
maximum SDR luminance.

It is guaranteed that SDR colors in this mode exactly match SDR colors in
non-HDR content on the page. For example, a canvas pixel value of `'(1,0,0)'` in
`'srgb'` is guaranteed to match the CSS color `'red'`.

Performing appropriate tone mapping is the responsibility of the browser, the
operating system, and the display device. If the `CanvasColorMetadata` specifies
a maximum luminance that is greater than the display's luminance, then tone
mapping will be applied to prevent clipping of luminance values below to the
specified maximum luminance. Note that the tone mapping algorithm may not alter
any SDR color values (otherwise the SDR color matching guarantee would be
violated).