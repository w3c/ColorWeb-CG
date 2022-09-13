# Canvas High Dynamic Range

## Proposal summary / TLDR

### Use cases

There are four main classes of High Dynamic Range (HDR) use that inform this proposal. They are:

* To draw HDR content with the minimum performance overhead.
* To draw HDR content in a way that will color match Standard Dynamic Range (SDR) content.
  * Such that SDR images drawn in the canvas will appear exactly as they would if display in an `<img>` tag.
* To display IUT-R BT.2100 Hybrid Log-Gamma (HLG) encoded HDR images and video in a canvas.
  * Such that HLG images drawn in the canvas will appear exactly the same as they would if displayed via an `<img>` or `<video>` tag.
* To display IUT-R BT.2100 Perceptual Quantiser (PQ) encoded HDR images and video in a canvas.
  * Such that PQ images drawn in the canvas will appear exactly the same as they would if displayed via an `<img>` or `<video>` tag.

### Proposed solution overview

The solution that we propose is to:

* Add attributes to the `ScreenAdvanced` interface to expose screen parameters relevant to high dynamic range and wide color gamut.
* Introduce new color spaces and precisions that are useful for HDR.
* Clearly define invertible and context-independent transformations between these spaces.
* Introduce the ability use more than 8 bits per pixel for a canvas element.
* Introduce the ability for an `HTMLCanvasElement` to configure HDR.

## Proposal

### Querying screen high dynamic range parameters

Add a new attribute to `ScreenAdvanced` to indicate the HDR headroom currently available on the screen.

```idl
  partial interface ScreenAdvanced {
    // The maximum luminance that the screen is capable of displaying across
    // the full area of the screen, as a multiple of the luminance of SDR white.
    // This will have a value of 1.0 for screens that are not HDR capable.
    readonly attribute double highDynamicRangeHeadroom;
  }
```

The high dynamic range headroom of a display is computed as:

```math
   HDR headroom = (HDR max luminance) / (SDR max luminance)
```

For a display device that is not HDR capable, this will have the value `1.0`.

### Querying screen wide color gamut parameters

Add new attributes to `ScreenAdvanced` to indicate the gamut and white point of the screen.

```idl
  partial interface ScreenAdvanced {
    // The color primaries and white point of the screen, in CIE 1931 xy
    // coordinates. These define the color gamut that the screen is capable of
    // displaying.
    readonly attribute double redPrimaryX;
    readonly attribute double redPrimaryY;
    readonly attribute double greenPrimaryX;
    readonly attribute double greenPrimaryY;
    readonly attribute double bluePrimaryX;
    readonly attribute double bluePrimaryY;
    readonly attribute double whitePointX;
    readonly attribute double whitePointY;
  }
```

### Enabling HDR on a canvas element

Add a new `CanvasHighDynamicRangeOptions` dictionary with HDR configuration options.

```idl
  dictionary CanvasHighDynamicRangeOptions {
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
    required float redPrimaryX;
    required float redPrimaryY;
    required float greenPrimaryX;
    required float greenPrimaryY;
    required float bluePrimaryX;
    required float bluePrimaryY;
    required float whitePointX;
    required float whitePointY;
    required float luminanceMin;
    required float luminanceMax;
  }
```

Add a new method to `HTMLCanvasElement` to allow configuring HDR.

```idl
  partial interface HTMLCanvasElement {
    bool configureHighDynamicRange(CanvasHighDynamicRangeOptions options);
  }
```

### Higher bit storage formats for 2D contexts

Add a new `CanvasStorageFormat` enum to allow for higher bit storage formats.

```idl
  enum CanvasStorageFormat {
    'unorm-8',
    'unorm-10-10-10-2',
    'float-16',
  }
```

Add a `CanvasStorageFormat` entry to `CanvasRenderingContext2DSettings` to allow 2D rendering contexts to specify their buffer format.

```idl
  partial dictionary CanvasRenderingContext2DSettings {
    CanvasStorageFormat storageFormat = "unorm-8";
  }
```

### Higher bit storage formats for WebGL and WebGPU

WebGL's proposed [``drawingBufferStorage``](https://github.com/KhronosGroup/WebGL/pull/3222) function allows for specifying higher bit depth formats.

WebGPU's ``GPUSwapChainDescriptor`` can allow for specifying higher bit depth formats.

### Tone mapping

As illustrated below, the proposal assumes that tone mapping, i.e. the rendering of an image with a given dynamic range onto a
display with different dynamic range, occurs in different parts of the system depending on the color space of the Canvas element:

* in the case where `rec2100-pq` are used, tone mapping is performed by the platform. This is akin to the
  scenario where the `src` of an `img` element is a PQ image.

* in the case where `rec2100-hlg` are used, tone mapping is performed by the display device. This is akin to the
  scenario where the `src` of an `img` element is an HLG image.

* in the case where any other color space is used (e.g, `srgb` or `srgb-linear`) tone mapping is performed by the web app, using display capabilities provided by the platform.

![Tone mapping scenarios](./tone-mapping-scenarios.png)

#### Suggested Tone mapping for HDR content on sRGB displays
Not these are only used to tonemap HDR images at the point of rendering for display when the display is known to be an sRGB display.  They are not used for conversion between colour spaces which is defined in section XXXX.

##### PQ signal

##### HLG signal

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

##### extended sRGB signal

Extended sRGB contains information that is outside of the capabilities of an sRGB monitor.  For this reason, the extended sRGB must first be converted for display.  
The extended sRGB signal can be converted for display on an sRGB monitor by first converting to HLG or PQ and then applying the relevant tonemapping to sRGB for displays.  For example:
```javascript
const [r_display, g_display, b_display] = tonemapREC2100HLGtoSRGBdisplay(
       convertExtendedSRGBtoREC2100HLG(r_extended_srgb, g_extended_srgb, b_extended_srgb));
```

### Color spaces

#### General

Update `PredefinedColorSpace` to include the following color spaces.

```idl
  partial enum PredefinedColorSpace {
    'srgb-linear',
    'rec2100-hlg',
    'rec2100-pq',
  }
```

#### srgb and display-p3

There already exist defined color spaces `srgb` and `display-p3`.

Note that the transfer function for these spaces [is already defined](https://www.w3.org/TR/css-color-4/#predefined) on all real numbers (not just the unit interval), as:

```javascript
  function electroOpticalTransferFunction(c) {
    let sign = c < 0? -1 : 1;
    let abs = Math.abs(c);
    if (abs <= 0.04045) {
      return = c / 12.92;
    }
    else {
      return = sign * (Math.pow((abs + 0.055) / 1.055, 2.4));
    }
  }
```

#### srgb-linear

This color space uses the same primaries as `srgb`, but with the identity function as the transfer function.

```javascript
  function electroOpticalTransferFunction(c) {
    return c;
  }
```

#### rec2100-hlg

The component signals are mapped to red, green and blue tristimulus values according to the Hybrid Log-Gamma (HLG) system specified in Rec. ITU-R BT.2100.

#### rec2100-pq

The component signals are mapped to red, green and blue tristimulus values according to the Perceptual Quantizer (PQ) system system specified in Rec. ITU-R BT.2100.

### Conversion between color spaces

#### General

There are several places in the HTML specification where a color space conversion is required (e.g, when [drawing images to a
canvas](https://html.spec.whatwg.org/multipage/canvas.html#colour-spaces-and-colour-correction), [retrieving image data from a
canvas](https://html.spec.whatwg.org/multipage/canvas.html#dom-context-2d-getimagedata), among others being added). There exists a
standard conversion that is applied to all data in these situations, namely, a conversion using relative colorimetric intent.

This operation is defined for [CSS Predefined Color Spaces](https://www.w3.org/TR/css-color-4/#predefined) in the HTML specification
[here](https://www.w3.org/TR/css-color-4/#predefined-to-lab).

In this section we define this conversion for the new predefined color spaces.

These conversions are expressed using a connection color space with the system colorimetry specified in Rec. ITU-R BT.2100:

* Red primary: `(0.708, 0.292)`
* Green chromaticity: `(0.170, 0.797)`
* Blue chromaticity: `(0.131, 0.046)`
* White chromaticity (D65): `(0.3127, 0.3290)`

_Note:_ The system colorimetry specified in Rec. ITU-R BT.2100 is identical to that specified in Rec. ITU-R BT.2020.

The conversion from color space A to color space B is performed according to the following steps:

* apply the inverse transfer function of color space A
* convert to the connection space by multiplying by the connection matrix of color space A
* convert to color space B by multiplying by by the inverse of the connection matrix of color space B
* apply any linear light scaling to correctly map the level of perceived diffuse white level in to color space B
* apply the transfer function of color space B

The domain and range of this conversion consist of all real values and the component signals in the connection color space are real numbers.

#### `srgb-linear`

* Transfer function: identity

* Connection matrix: matrix to convert the [sRGB primaries](https://www.w3.org/TR/css-color-4/#valdef-color-srgb) to the primary colours specified in Rec. ITU-R BT.2100

* Linear light scaling: 1.0

#### `srgb`

* Transfer function: See [srgb-linear](#srgb-linear)

* Connection matrix: matrix to convert the [sRGB primaries](https://www.w3.org/TR/css-color-4/#valdef-color-srgb) to the primary colours specified in Rec. ITU-R BT.2100

* Linear light scaling: 1.0

_Note:_ The domain of this transformation function is all real values. Its domain is not restricted to the unit interval [0, 1].

Also see note in the Issues section at the bottom about whether this space should be distinct from the existing `'srgb'` space.

#### `rec2100-hlg`


* Transfer function: The HLG Reference inverse OETF specified at Rec. ITU-R BT.2100, scaled

```javascript
  function connectionTransferFunction(x) {
    const a = 0.17883277;
    const b = 1 - 4*a;
    const c = 0.5 - a * Math.log(4 * a);
    const k = (Math.exp((0.75 - c) / a) + b) / 12;
    if (x < 0) {
      return 0;
    } else if (x <= 0.5) {
      return x * x / (3 * k);
    } else if (x <= 1) {
      return (Math.exp((v - c) / a) + b) / (12 *  k);
    } else {
      return 1 / scale;
    }
```

_Note:_ the factor `k` is selected such that a signal of `0.75` results in a linear color value of 1 in the connection color space.

* Connection matrix: Identity

#### `rec2100-pq`

* Transfer function: The inverse of the Reference PQ EOTF specified at Rec. ITU-R BT.2100, scaled

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

_Note:_ The factor `k` is selected such that a display luminance of 203 cd/m<sup>2</sup> results in a linear color value of 1 in the connection color space.


#### Conversions via Color Connection Space

##### Conversion from extended-sRGB to HLG

_Input:_ Full-range non-linear floating-point `extended-srgb` pixel with black at 0.0 and diffuse white at 1.0. Values may exist outside the range 0.0 to 1.0.

_Output:_ Full-range non-linear floating-point `rec2100-hlg` pixel with black at 0.0 and diffuse white at 0.75. Values may exist outside the range 0.0 to 1.0.

_Process:_
 1. Linearize using the SRGB EOTF
 2. Convert from `extended-srgb` color space to `rec2100-hlg` color space
 3. Scale pixel values - See Note 1
 4. Apply HLG Inverse EOTF to convert to HLG from Pseudo-Display Light with Lw = 302 cd/m2 - See Note 1
  * apply HLG Inverse OOTF
  * apply HLG OETF - See Note 2

_Note 1:_ As `rec2100-hlg` is a relative format, the brightness of the virtual monitor used for mathematical transforms can be chosen to be any level. In this transform it is chosen to be 302 cd/m2 so that the brightness of diffuse white matches the diffuse white of sRGB (80 cd/m2). Using the extended range gamma formula in footnote 2 of BT.2100, this also sets the HLG Inverse OOTF to be unity. The value 0.265 is calculated by taking the inverse OETF of 0.75, the `rec2100-hlg` diffuse white level.

_Note 2:_ See section 5.3 in ITU-R BT.2408-4 relating to negative transfer functions in format conversions.

```javascript
function convertExtendedSRGBtoREC2100HLG(r, g, b) {
  const systemGamma = 1.0;
  const linearLightScaler = 0.26496256042100724;

  const r1 = srgb_eotf(r);
  const g1 = srgb_eotf(g);
  const b1 = srgb_eotf(b);

  const [r2, g2, b2] = matrixXYZtoBT2020(matrixSRGBtoXYZ(r1,g1,b1));

  const r3 = linearLightScaler * r2;
  const g3 = linearLightScaler * g2;
  const b3 = linearLightScaler * b2;

  const [r4, g4, b4] = hlg_inverse_ootf(r3, g3, b3, systemGamma);
  const [r5, g5, b5] = hlg_oetf(r4, g4, b4);

  return [r5, g5, b5]
}
```

##### Conversion from extended-linear-sRGB to HLG

_Input:_ Full-range non-linear floating-point `extended-linear-srgb` pixel with black at 0.0 and diffuse white at 1.0. Values may exist outside the range 0.0 to 1.0.

_Output:_ Full-range non-linear floating-point `rec2100-hlg` pixel with black at 0.0 and diffuse white at 0.75. Values may exist outside the range 0.0 to 1.0.

_Process:_
1. Convert from `extended-linear-srgb` color space to `rec2100-hlg` color space
2. Scale pixel values - See Note 1
3. Apply HLG Inverse EOTF to convert to HLG from Pseudo-Display Light with Lw = 302 cd/m2 - See Note 1
 * apply HLG Inverse OOTF
 * apply HLG OETF - See Note 2

```javascript
function convertExtendedLinearSRGBtoREC2100HLG(r, g, b) {
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

##### Conversion from extended-sRGB to PQ

_Input:_ Full-range non-linear floating-point `extended-srgb` pixel with black at 0.0 and diffuse white at 1.0. Values may exist outside the range 0.0 to 1.0.

_Output:_ Full-range non-linear floating-point `rec2100-pq` pixel with black at 0.0 and diffuse white at ???. Values may exist outside the range 0.0 to 1.0.

_Process:_

##### Conversion from extended-sRGB to PQ

_Input:_ Full-range non-linear floating-point `extended-linear-srgb` pixel with black at 0.0 and diffuse white at 1.0. Values may exist outside the range 0.0 to 1.0.

_Output:_ Full-range non-linear floating-point `rec2100-pq` pixel with black at 0.0 and diffuse white at ???. Values may exist outside the range 0.0 to 1.0.

_Process:_

##### Conversion from HLG to extended-sRGB

_Input:_ Full-range non-linear floating-point `rec2100-hlg` pixel with black at 0.0 and diffuse white at 0.75. Values may exist outside the range 0.0 to 1.0.

_Output:_ Full-range non-linear floating-point `extended-srgb` pixel with black at 0.0 and diffuse white at 1.0. Values may exist outside the range 0.0 to 1.0.

_Process:_
1. Apply HLG EOTF to convert the non-linear `rec2100-hlg` Signal to linear Pseudo-Display Light with Lw = 302 cd/m2 - See Note 1
 * apply inverse HLG OETF
 * apply HLG OOTF to derive linear display light
2. Scale pixel values
3. Convert from ITU BT.2100-1 color space to SRGB color space
4. Convert to non-linear SRGB using the SRGB Inverse EOTF


```javascript
function convertREC2100HLGtoExtendedSRGB(r, g, b) {
  const systemGamma = 1.0;
  const linearLightScaler = 1.0 / 0.26496256042100724;

  const [r1, g1, b1] = hlg_inverse_oetf(r, g, b);
  const [r2, g2, b2] = hlg_ootf(r1, g1, b1, systemGamma);

  const r3 = linearLightScaler * r2;
  const g3 = linearLightScaler * g2;
  const b3 = linearLightScaler * b2;

  const [r4, g4, b4] = matrixXYZtoSRGB(matrixBT2020toXYZ(r3, g3, b3));
  const r5 = srgb_inverse_eotf(r4);
  const g5 = srgb_inverse_eotf(g4);
  const b5 = srgb_inverse_eotf(b4);

  return [r5, g5, b5];
}
```

##### Conversion from HLG to extended-linear-sRGB

  _Input:_ Full-range non-linear floating-point `rec2100-hlg` pixel with black at 0.0 and diffuse white at 0.75. Values may exist outside the range 0.0 to 1.0.

  _Output:_ Full-range non-linear floating-point `extended-linear-srgb` pixel with black at 0.0 and diffuse white at 1.0. Values may exist outside the range 0.0 to 1.0.

  _Process:_
  1. Apply HLG EOTF to convert the non-linear `rec2100-hlg` Signal to linear Pseudo-Display Light with Lw = 302 cd/m2 - See Note 1
   * apply inverse HLG OETF
   * apply HLG OOTF to derive linear display light
  2. Scale pixel values
  3. Convert from ITU BT.2100-1 color space to SRGB color space

```javascript
function convertREC2100HLGtoExtendedLinearSRGB(r, g, b) {
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


##### Conversion from PQ to extended-sRGB

  _Input:_ Full-range non-linear floating-point `rec2100-pq` pixel with black at 0.0 and diffuse white at ???. Values may exist outside the range 0.0 to 1.0.

  _Output:_ Full-range non-linear floating-point `extended-srgb` pixel with black at 0.0 and diffuse white at 1.0. Values may exist outside the range 0.0 to 1.0.

  _Process:_

##### Conversion from PQ to extended-linear-sRGB

  _Input:_ Full-range non-linear floating-point `rec2100-pq` pixel with black at 0.0 and diffuse white at ???. Values may exist outside the range 0.0 to 1.0.

  _Output:_ Full-range non-linear floating-point `extended-linear-srgb` pixel with black at 0.0 and diffuse white at 1.0. Values may exist outside the range 0.0 to 1.0.

  _Process:_


### Compositing the HDR `HTMLCanvasElement`

The compositing behavior of an `HTMLCanvasElement` may be specified using the `configureHighDynamicRange` method.

#### Default behavior

This section describes the default compositing behavior for canvas element.
This is the behavior that happens if `configureHighDynamicRange` is not called, or if is called specifying `'default'` as the mode.

In this mode, the `HTMLCanvasElement` will be composited exactly as an `<img>` or `<video>` element with a source in the canvas' color space would be composited.

This means that if the canvas' color space is `'rec2100-hlg'` or `'rec2100-pq'`, then the canvas will be composited using high dynamic range, where available.

Note that if the canvas' color space is `'srgb-linear'` or `'srgb'`, then the canvas will not be composited using high dynamic range. Pixel values outside of the [0, 1] interval will extend the displayed gamut beyond sRGB, but not the displayed luminance beyond the maximum SDR luminance.

Performing appropriate tone mapping is the responsibility of the browser, the operating system, and the display device.
If the `configureHighDynamicRange` method is called with `CanvasHighDynamicRangeOptions` that specify metadata and the canvas' color space is `'rec2100-pq'`, then this metadata will be interpreted during compositing in the same way that it would be interpreted if included in a source displayed via an `<img>` or `<video>` element.

#### Extended mode

If the canvas' color space is `'srgb-linear'` or `'srgb'`, then pixels values outside of the [0, 1] interval will extend the displayed luminance beyond the maximum SDR luminance.

It is guaranteed that SDR colors in this mode exactly match SDR colors in non-HDR content on the page.
For example, a canvas pixel value of `'(1,0,0)'` in `'srgb'` is guaranteed to match the CSS color `'red'`.

Performing appropriate tone mapping is the responsibility of the browser, the operating system, and the display device.
If the `configureHighDynamicRange` method is called with `CanvasHighDynamicRangeOptions` that specify a maximum luminance that is greater than the display's luminance, then tone mapping will be applied to prevent clipping of luminance values below to the specified maximum luminance.
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
    canvas.configureHighDynamicRange({mode:'extended'});

    var gl = canvas.getContext('webgl2');
    gl.drawingBufferStorage(gl.RGBA16F, canvas.width, canvas.height);
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
    canvas.configureHighDynamicRange({mode:'hlg'});
    var context = canvas.getContext('2d',
        {colorSpace:'srgb-linear', storageFormat:'float-16'});
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
