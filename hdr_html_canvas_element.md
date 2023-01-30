# High Dynamic Range (HDR) HTML Canvas

## Scope

We propose to extend the HTML Canvas API to support High Dynamic Range (HDR) imagery.

## Primary requirements

* Minimum performance overhead
* Draw Standard Dynamic Range (SDR) images such that these images appear as they would if display in an `<img>` tag
* Draw IUT-R BT.2100 Hybrid Log-Gamma (HLG) encoded (HDR) images such that these images appear as they would if displayed via an
  `<img>` or `<video>` tag
* Draw IUT-R BT.2100 Perceptual Quantiser (PQ) encoded (HDR) images such that these images appear as they would if displayed via an
  `<img>` or `<video>` tag

## Summary

We propose to introduce:

* three new HDR-specific color spaces: `srgb-linear`, `rec2100-hlg`, `rec2100-pq`
* an HDR mode for the existing `srgb` color space
* pixel bit depths greater than 8 bits
* image parameters relevant to HDR rendering to the `HTMLCanvasElement`'s rendering context.
* screen parameters relevant to HDR rendering using the `ScreenAdvanced` interface
* fallback reference conversions between HDR color spaces
* fallback reference conversion between SDR and HDR color spaces

## Proposal

### Querying screen HDR parameters

Add a new attribute to `ScreenAdvanced` to indicate the HDR headroom currently available on the screen.

```idl
  partial interface ScreenAdvanced {
    // The maximum luminance that the screen is capable of displaying across
    // the full area of the screen, as a multiple of the luminance of SDR white.
    // This will have a value of 1.0 for screens that are not HDR capable.
    readonly attribute double highDynamicRangeHeadroom;
  }
```

See below for more details on the display of floating point color values.

### Querying screen wide color gamut parameters

Add new attributes to `ScreenDetailed` to indicate the gamut and white point of the screen.

```idl
  partial interface ScreenDetailed {
    // The color volume that the screen is capable of displaying.
    readonly attribute ColorVolume colorVolume;
  }
```

### Color volume structure

Add a new color volume structure for use in several places.

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

### Enabling HDR on a canvas element's rendering context

Add a new `CanvasColorMetadata` dictionary with HDR configuration options.

```idl
  dictionary CanvasColorMetadata {
    CanvasHighDynamicRangeMode mode = 'default';
    CanvasSmpteSt2086Metadata smpteSt2086Metadata;
  }

  enum CanvasHighDynamicRangeMode {
    // The default behavior. Enables HDR for 'rec2100-hlg' and 'rec2100-pq'
    // color spaces only.
    'default',

    // Enables extended luminance while preserving SDR color matching.
    'extended',
  }

  // SMPTE ST 2086 color volume metadata.
  dictionary CanvasSmpteSt2086Metadata {
    ColorVolume colorVolume;
    required float minimumLuminanceNits;
    required float maximumLuminanceNits;
  }
```

Add a mechanism for specifying this on `CanvasRenderingContext2D`, `OffscreenCanvasRenderingContext2D`, `WebGLRenderingContextBase`, and `GPUCanvasContext`.

For 2D canvas this would be:
```idl
  partial interface CanvasRenderingContext2D/OffscreenCanvasRenderingContext2D {
    attribute CanvasColorMetadata colorMetadata;
  }
```

For WebGL this would be:
```idl
  partial interface WebGLRenderingContextBase {
    attribute CanvasColorMetadata drawingBufferColorMetadata;
  }
```

For WebGPU this would be:
```idl
  partial interface GPUCanvasContext {
    attribute CanvasColorMetadata colorMetadata;
  }
```

### Pixel bit depths greater than 8 bits for Canvas, WebGL and WebGPU

As implied by its names, HDR imagery requires more then 8 bits per component. Proposals to extend the HTML Canvas API and WebGL APIs are covered in other documents:

* Canvas 2D's proposed [Canvas Floating Point Color Values](https://github.com/w3c/ColorWeb-CG/blob/main/canvas_float.md) allows for
  specifying higher bit depth canvas and ImageData.
* WebGL's proposed [``drawingBufferStorage``](https://github.com/KhronosGroup/WebGL/pull/3222) function allows for specifying higher
  bit depth formats.

[The WebGPU ``GPUCanvasConfiguration``](https://www.w3.org/TR/webgpu/#canvas-configuration) already allows 16-bit float values and
can be further extended if needs be.

### Extend the existing `srgb` color space for HDR imagery

Define a *screen's native linear color space* to be the color space with color primaries and white point set to the screen's
`ScreenDetailed`'s `colorVolume` and the identity as its transfer function.

A color is said to be within a screen's *default* range if, when that color is converted to the screen's native linear color space
(using relative colorimetric intent), all of its color components are within the `[0, 1]` interval. When displaying content produced
by a rendering context with `CanvasHighDynamicRangeMode` set to `'default'`, all colors that are within the screen's default range
must be displayed without any clamping or roll-offs (neither in chrominance nor luminance).

A color is said to be within a screen's *extended* range if, when that color is converted to the screen's native linear color space
(using relative colorimetric intent), all of its color components are within the `[0, highDynamicRangeHeadroom]` interval. When
displaying content produced by a rendering context with `CanvasHighDynamicRangeMode` set to `'extended'`, all colors that are within
the screen's extended range must be displayed without any clamping or roll-offs (neither in chrominance nor luminance).

Colors that do not fall under this guarantee should be clamped to the indicated range during display, but may be subject to screen-specific behavior.

### New HDR-specific color spaces

#### General

Update `PredefinedColorSpace` to include the following color spaces.

```idl
  partial enum PredefinedColorSpace {
    'srgb-linear',
    'rec2100-hlg',
    'rec2100-pq',
  }
```

As illustrated below, the proposal assumes that tone mapping, i.e. the rendering of an image with a given dynamic range onto a
display with different dynamic range, occurs in different parts of the system depending on the color space of the Canvas element:

* in the case where `rec2100-pq` are used, tone mapping is performed by the platform. This is akin to the scenario where the `src`
  of an `img` element is a PQ image.

* in the case where `rec2100-hlg` are used, tone mapping is performed by the display device. This is akin to the scenario where the
  `src` of an `img` element is an HLG image.

* in the case where any other color space is used (e.g, `srgb` or `srgb-linear`) tone mapping is performed by the web app, using
  display capabilities provided by the platform.

![Tone mapping scenarios](./tone-mapping-scenarios.png)

#### srgb-linear

This color space uses the same primaries and white point as `srgb`, but with the identity function as the transfer function.

```javascript
  function electroOpticalTransferFunction(c) {
    return c;
  }
```

#### rec2100-hlg

The component signals are mapped to red, green and blue tristimulus values according to the Hybrid Log-Gamma (HLG) system specified in Rec. ITU-R BT.2100.

#### rec2100-pq

The component signals are mapped to red, green and blue tristimulus values according to the Perceptual Quantizer (PQ) system system specified in Rec. ITU-R BT.2100.

### Color space conversions

#### Background

In general, application should avoid conversions between color spaces and maintain imagery in its original color space: conversions
between color spaces are not necessarily reversible and do not necessarily result in the same image appearance. In particular,
conversion of an HDR image to an SDR will result in a signification loss of information and an SDR image that is different from the
SDR image that would have been mastered from the same source material. From that perspective, converting from HDR to SDR imagery is
no different than converting RGBA images to 16-color pallette images.

Nevertheless, the HTML specification allows color space conversion in several scenarios, e.g., when [drawing images to a
canvas](https://html.spec.whatwg.org/multipage/canvas.html#colour-spaces-and-colour-correction), [retrieving image data from a
canvas](https://html.spec.whatwg.org/multipage/canvas.html#dom-context-2d-getimagedata), among others being added). The conversions
between predefined SDR color spaces are defined at <https://www.w3.org/TR/css-color-4/>, and this proposal similarly defines
conversions for HDR color spaces.

These conversions fall into two broad categories:

* conversion between HDR color spaces
* conversion between an HDR and an SDR color space (tone mapping)

#### Between HDR color spaces

##### General

Conversions to and from extended `srgb` are not included since extended `srgb` is trivially converted to and from `srgb-linear`.

Conversions between `rec2100-hlg` and `rec2100-pq` are implicitly specified through conversions to and from `srgb-linear`.

The domain and range of the conversions consist of all real values.

##### `rec2100-hlg` to `srgb-linear`

_Input:_ Full-range non-linear floating-point `rec2100-hlg` pixel with black at 0.0 and diffuse white at 0.75. Values may exist outside the range 0.0 to 1.0.

_Output:_ Full-range non-linear floating-point `srgb-linear` pixel with black at 0.0 and diffuse white at 1.0. Values may exist outside the range 0.0 to 1.0.

_Process:_

1. Apply HLG EOTF to convert the non-linear `rec2100-hlg` Signal to linear Pseudo-Display Light with Lw = 302 cd/m2 - See Note 1
    * apply inverse HLG OETF
    * apply HLG OOTF to derive linear display light
2. Scale pixel values
3. Convert from ITU BT.2100-1 color to SRGB color

```javascript
function convertREC2100HLGtoLinearSRGB(r, g, b) {
  const systemGamma = 1.0;
  const linearLightScaler = 1.0 / 0.26496256042100724;

  const [r1, g1, b1] = hlg_inverse_oetf(r, g, b);
  const [r2, g2, b2] = hlg_ootf(r1, g1, b1, systemGamma);

  const r3 = linearLightScaler * r2;
  const g3 = linearLightScaler * g2;
  const b3 = linearLightScaler * b2;

  const [r4, g4, b4] = matrixXYZtoSRGB(matrixBT2020toXYZ(r3, g3, b3));

  return [r4, g4, b4];
}
```

##### `srgb-linear` to `rec2100-hlg`

_Input:_ Full-range non-linear floating-point `srgb-linear` pixel with black at 0.0 and diffuse white at 1.0. Values may exist outside the range 0.0 to 1.0.

_Output:_ Full-range non-linear floating-point `rec2100-hlg` pixel with black at 0.0 and diffuse white at 0.75. Values may exist outside the range 0.0 to 1.0.

_Process:_

1. Convert from `extended-srgb-linear` color to `rec2100-hlg` color
2. Scale pixel values - See Note 1
3. Apply HLG Inverse EOTF to convert to HLG from Pseudo-Display Light with Lw = 302 cd/m2 - See Note 1
    * apply HLG Inverse OOTF
    * apply HLG OETF - See Note 2

```javascript
function convertLinearSRGBtoREC2100HLG(r, g, b) {
  const systemGamma = 1.0;
  const linearLightScaler = 0.26496256042100724;

  const [r2, g2, b2] = matrixXYZtoBT2020(matrixSRGBtoXYZ(r1, g1, b1));

  const r3 = linearLightScaler * r2;
  const g3 = linearLightScaler * g2;
  const b3 = linearLightScaler * b2;

  const [r4, g4, b4] = hlg_inverse_ootf(r3, g3, b3, systemGamma);
  const [r5, g5, b5] = hlg_oetf(r4, g4, b4);

  return [r5, g5, b5];
}
```

_Note 1:_ As `rec2100-hlg` is a relative format, the brightness of the virtual monitor used for mathematical transforms can be chosen to be any level. In this transform it is chosen to be 302 cd/m2 so that the brightness of diffuse white matches the diffuse white of sRGB (80 cd/m2). Using the extended range gamma formula in footnote 2 of BT.2100, this also sets the HLG Inverse OOTF to be unity. The value 0.265 is calculated by taking the inverse OETF of 0.75, the `rec2100-hlg` diffuse white level.

_Note 2:_ See section 5.3 in ITU-R BT.2408-4 relating to negative transfer functions in format conversions.

##### `srgb-linear` to `rec2100-pq`

_Input:_ Full-range non-linear floating-point `srgb-linear` pixel with black at 0.0 and diffuse white at 1.0. Values may exist outside the range 0.0 to 1.0.

_Output:_ Full-range non-linear floating-point `rec2100-pq` pixel with black at 0.0 and diffuse white at ???. Values may exist outside the range 0.0 to 1.0.

_Process:_

```javascript
  function connectionTransferFunction(x) {
    const c1 =  107 / 128;
    const c2 = 2413 / 128;
    const c3 = 2392 / 128;
    const m1 = 1305 / 8192;
    const m2 = 2523 / 32;
    const k = 203;
    if (x < 0) {
      return 0;
    } else if (x <= 1) {
      const p = Math.pow(x, 1 / m2);
      return (10000 / k) * Math.pow((p - c1) / (c2 - c3 * p), 1 / m1);
    } else {
      return (10000 / k);
    }
```

_Note:_ The factor `k` is selected such that a display luminance of 203 cd/m<sup>2</sup> results in a linear color value of 1 in `srgb-linear`.

##### `srgb-linear` to `rec2100-pq`

_Input:_ Full-range non-linear floating-point `extended-srgb` pixel with black at 0.0 and diffuse white at 1.0. Values may exist outside the range 0.0 to 1.0.

_Output:_ Full-range non-linear floating-point `rec2100-pq` pixel with black at 0.0 and diffuse white at ???. Values may exist outside the range 0.0 to 1.0.

_Process:_

#### Between SDR and HDR color spaces

##### General

Conversions to and from `srgb` are provided for `rec2100-pq` and `rec2100-hlg` color spaces. Extended `srgb` images can be converted
for display on an sRGB monitor by first converting to `rec2100-pq` and `rec2100-hlg` first and then applying the relevant conversion
to `srgb`.

##### `rec2100-pq` to `srgb`

See Annex B at SMPTE ST 2094-10.

##### `rec2100-hlg` to `srgb`

_Input:_ Full-range non-linear floating-point `rec2100-hlg` pixel with black at 0.0 and diffuse white at 0.75. Values may exist outside the range 0.0 to 1.0.

_Output:_ Full-range non-linear floating-point `srgb` pixel with black at 0.0 and diffuse white at 1.0. Values may exist outside the range 0.0 to 1.0.

_Process:_
  1. Pseudo-linearize the HLG signal exploiting its backwards compatibility with SDR consumer displays
  2. Convert from ITU BT.2100 color space to sRGB color space
  3. Convert back to non-linear using a reciprocal transform

_Note 3_ This transform utilises the backwards compatibility of ITU-R BT.2100 HLG HDR with consumer electronic displays.  Prior to display, the gamut may need to be limited to the range 0-1.  The simplest method is to clip values but other gamut reduction techniques may provide better output images.

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

##### `srgb` to `rec2100-hlg`

See [TTML 2, Annex Q.2, steps 1-8](https://www.w3.org/TR/ttml2/#hlg-hdr) with `tts:luminanceGain = 203/80`.

##### `srgb` to `rec2100-pq`

See [TTML 2, Annex Q.1, steps 1-8](https://www.w3.org/TR/ttml2/#hdr-compositing).

### Compositing the HDR `HTMLCanvasElement`

The compositing behavior of an `HTMLCanvasElement` may be specified via the `CanvasColorMetadata` attribute of its rendering context.

#### Default behavior

This section describes the default compositing behavior for canvas element.
This is the behavior that happens if the `CanvasColorMetadata` is not set, or if is called specifying `'default'` as the mode.

In this mode, the `HTMLCanvasElement` will be composited exactly as an `<img>` or `<video>` element with a source in the canvas' color space would be composited.

This means that if the canvas' color space is `'rec2100-hlg'` or `'rec2100-pq'`, then the canvas will be composited using high dynamic range, where available.

Note that if the canvas' color space is `'srgb-linear'` or `'srgb'`, then the canvas will not be composited using high dynamic range. Pixel values outside of the [0, 1] interval will extend the displayed gamut beyond sRGB, but not the displayed luminance beyond the maximum SDR luminance.

Performing appropriate tone mapping is the responsibility of the browser, the operating system, and the display device.
If the `CanvasColorMetadata` is set to specify metadata and the canvas' color space is `'rec2100-pq'`, then this metadata will be interpreted during compositing in the same way that it would be interpreted if included in a source displayed via an `<img>` or `<video>` element.

#### Extended mode

If the canvas' color space is `'srgb-linear'` or `'srgb'`, then pixels values outside of the [0, 1] interval will extend the displayed luminance beyond the maximum SDR luminance.

It is guaranteed that SDR colors in this mode exactly match SDR colors in non-HDR content on the page.
For example, a canvas pixel value of `'(1,0,0)'` in `'srgb'` is guaranteed to match the CSS color `'red'`.

Performing appropriate tone mapping is the responsibility of the browser, the operating system, and the display device.
If the `CanvasColorMetadata` specifies a maximum luminance that is greater than the display's luminance, then tone mapping will be applied to prevent clipping of luminance values below to the specified maximum luminance.
Note that the tone mapping algorithm may not alter any SDR color values (otherwise the SDR color matching guarantee would be violated).

## Privacy considerations

The precise capabilities of the output display, such as the exact color gamut and the precise maximum luminance values, are fingerprinting vectors, and should not be exposed to the application without user permission.

Access to these values is protected behind the same permission prompt that protects other screen capability fingerprinting vectors in the [window placement proposal](https://github.com/webscreens/window-placement/blob/main/EXPLAINER.md).

## Example Applications

### Detecting HDR capabilities

The `getScreens` method will prompt the user for permission to reveal fingerprintable information.

```javascript
  let screens = await window.getScreens();
  console.log('HDR headroom: ' + currentScreen.highDynamicRangeHeadroom);
```

If `getScreens` is denied, then media queries may be used to determine the screen's capabilities, at a high level.

```javascript
  let hasHighDynamicRange = window.matchMedia('(dynamic-range: high)').matches;
  console.log('HDR capable: ' + hasHighDynamicRange);
```

### The `srgb-linear` color space

#### WebGL using extended mode

In this example, a WebGL application enables and HDR default drawing buffer, and clears it to the pixel value `(1,1,1,1)`.

```javascript
    var canvas = document.getElementById('MyCanvas');

    var gl = canvas.getContext('webgl2');
    gl.drawingBufferStorage(gl.RGBA16F, canvas.width, canvas.height);
    gl.drawingBufferColorMetadata = {mode:'extended'};
    gl.colorSpace = 'srgb-linear';
    gl.clearColor(1.0, 1.0, 1.0, 1.0);
    gl.clear(gl.COLOR_BUFFER_BIT);
```

When composited, this canvas is guaranteed to be the same color as the CSS color `'white'`.

### The `rec2100-hlg` color space

#### Displaying an HLG image in an SDR 2D canvas

In this example, we use an SDR 2D canvas to display a HLG image.
This image will map into the SDR range without clipping.

```javascript
    var canvas = document.getElementById('MyCanvas');
    var context = canvas.getContext('2d');

    var image = new Image();
    image.onload = function() {
      context.drawImage(image, 0, 0, image.width, image.height);
    }
    var url = 'https://storage.googleapis.com/dalecurtis/cosmos_hlg.avif';
    image.src = url;
```

#### Displaying an HLG image in an HLG 2D canvas

In this example, we use an HLG 2D canvas to display a HLG image.

```javascript
    var canvas = document.getElementById('MyCanvas');
    var context = canvas.getContext('2d',
        {colorSpace:'rec2100-hlg', storageFormat:'unorm-10-10-10-2'});

    // Load and draw the image to the canvas.
    var image = new Image();
    image.onload = function() {
      context.drawImage(image, 0, 0, image.width, image.height);
    }
    var url = 'https://storage.googleapis.com/dalecurtis/cosmos_hlg.avif';
    image.src = url;
```

This canvas, when composited, will always be identical to displaying the image an ordinary `<img>` element.

```xml
  <img src='https://storage.googleapis.com/dalecurtis/cosmos_hlg.avif'/>
```

#### Adding subtitles to an HLG image

Suppose we wish to change the above example to draw subtitles at a brightness that corresponds to an HLG signal value of 0.75.

```javascript
    var canvas = document.getElementById('MyCanvas');
    var context = canvas.getContext('2d',
        {colorSpace:'rec2100-hlg', storageFormat:'unorm-10-10-10-2'});

    // Load and draw the image to the canvas.
    var image = new Image();
    image.onload = function() {
      context.drawImage(image, 0, 0, image.width, image.height);

      // Draw a subtitle!
      context.fillStyle = 'color(rec2100-hlg 0.75  0.75 0.75)';
      context.fillText('Hello, I am a subtitle!');
    }
    var url = 'https://storage.googleapis.com/dalecurtis/cosmos_hlg.avif';
    image.src = url;
```

Note that this is identical to specifying the subtitle color in `'srgb-linear'`.

```javascript
    context.fillStyle = 'color(srgb-linear 0.265  0.265 0.265)';
```

### The `rec2100-pq` color space

In this example, we use a PQ 2D canvas to display a PQ image.

```javascript
    var canvas = document.getElementById('MyCanvas');
    var context = canvas.getContext('2d',
        {colorSpace:'rec2100-pq', storageFormat:'unorm-10-10-10-2'});

    // Load and draw the image to the canvas.
    var image = new Image();
    image.onload = function() {
      context.drawImage(image, 0, 0, image.width, image.height);
    }
    var url = 'https://storage.googleapis.com/dalecurtis/cosmos_1000_pq_hdr.avif';
    image.src = url;
```

There is no guarantee that this canvas, when composited, will be equivalent to displaying the image an ordinary `<img>` element.

```xml
  <img src='https://storage.googleapis.com/dalecurtis/cosmos_hlg.avif'/>
```

The reason this guarantee cannot be made is that the `<img>` element may do custom tonemapping based on embedded metadata.
There does not exist any API for extracting this metadata from an `Image` object, and thus this metadata cannot be passed on to the `HTMLCanvasElement`.

## Issues

The above is a simplified version of the API that was proposed earlier.

### HDR compositing independent of color space

In the existing API, there is no way to have a linear space working space for an HLG or PQ canvas

The fix for this that I propose is to allow a `'hlg'` and `'pq'` `CanvasHighDynamicRangeMode`.

In that case, the code to work in a linearized HLG space would be:

```javascript
    var context = canvas.getContext('2d',
        {colorSpace:'srgb-linear', storageFormat:'float-16'});
    context.colorMetadata = {mode:'hlg'};
```

### ImageBitmap conversion options

The `ImageBitmapOptions` structure already has `colorSpace` and `colorSpaceConversion` members.

The `colorSpaceConversion` has `default`, which is relative colorimetric intent, and `none`, which is to simply reinterpret values directly.

We could consider adding a `perceptual` option for `colorSpaceConversion`, which would perform some sort of tonemapping. We could also add a `bt2408` option.

### Appropriate location for HDR configuration

In this proposal, the HDR configuration data has been attached to the `HTMLCanvasElement`.
Arguably, the HDR configuration data could be attached to the `CanvasRenderingContext2D` and `WebGLRenderingContextBase`.

The HDR configuration data should travel with an `ImageBitmap` when displayed in an `ImageBitmapRenderingContext`.
That may inform where we should put this.
