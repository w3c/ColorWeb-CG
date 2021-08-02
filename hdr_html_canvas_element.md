# Canvas High Dynamic Range

## Proposal summary / TLDR

### Use cases

There are four main classes of HDR use that inform this proposal. They are:

* To draw HDR content with the minimum performance overhead.
* To draw HDR content in a way that will color match SDR content.
  * Such that SDR images drawn in the canvas will appear exactly as they would if display in an `<img>` tag.
* To display HLG encoded HDR images and video in a canvas.
  * Such that HLG images drawn in the canvas will appear exactly the same as they would if displayed via an `<img>` or `<video>` tag.
* To display PQ encoded HDR images and video in a canvas.
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
    // TODO for v2: Add metadata parameters.
  }

  enum CanvasHighDynamicRangeMode {
    // The default behavior. Enables HDR for 'rec2100-hlg' and 'rec2100-pq'
    // color spaces only.
    'default',

    // Enables extended luminance while preserving SDR color matching for
    // 'extended-linear-srgb' and 'extended-linear-srgb' color spaces.
    'extended',
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

* in the case where `rec2100-hlg` or `rec2100-pq` are used, tone mapping is performed by the platform. This is akin to the
  scenario where the `src` of an `img` element is a PQ or HLG image.

* in the case where `extended-linear-srgb` or `extended-srgb` are used, tone mapping is performed by the web app, using display capabilities provided by the platform.

![Tone mapping scenarios](./tone-mapping-scenarios.png)

### Color spaces

#### General

Update `PredefinedColorSpace` to include the following color spaces.

```idl
  partial enum PredefinedColorSpace {
    'extended-linear-srgb',
    'extended-srgb',
    'rec2100-hlg',
    'rec2100-pq',
  }
```

#### extended-srgb

The component signals are mapped to red, green and blue tristimulus values according to the following:

* Red primary chromaticity: `(0.640, 0.330)`
* Green primary chromaticity: `(0.300, 0.600)`
* Blue primary chromaticity: `(0.150, 0.060)`
* White point chromaticity: `(0.3127, 0.3290)`
* Transfer function:

```
   E = | E' / 12.92, if abs(E') ≤ 0.04045
       | ((E' + 0.055) / 1.055)^2.4, otherwise

       with E' ∈ ℝ
```

where `E'` is the non-linear colour value and `E` is the linear colour value

#### extended-linear-srgb

The component signals are mapped to red, green and blue tristimulus values according to the following:

* Red primary chromaticity: `(0.640, 0.330)`
* Green primary chromaticity: `(0.300, 0.600)`
* Blue primary chromaticity: `(0.150, 0.060)`
* White point chromaticity: `(0.3127, 0.3290)`
* Transfer function: `E = E', with E' ∈ ℝ` where `E'` is the non-linear colour value and `E` is the linear colour value

#### rec2100-hlg

The component signals are mapped to red, green and blue tristimulus values according to the Hybrid Log-Gamma (HLG) system specified in Rec. ITU-R BT.2100.

#### rec2100-pq

The component signals are mapped to red, green and blue tristimulus values according to the PQ system system specified in Rec. ITU-R BT.2100.

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
* White chromaticity: `(0.3127, 0.3290)`

_Note:_ The system colorimetry specified in Rec. ITU-R BT.2100 is identical to that specified in Rec. ITU-R BT.2020.

The conversion from color space A to color space B is performed according to the following steps:

* apply the inverse transfer function of color space A
* convert to the connection space by multiplying by the connection matrix of color space A
* convert to color space B by multiplying by by the inverse of the connection matrix of color space B
* apply the transfer function of color space B

The domain and range of this conversion consist of all real values and the component signals in the connection color space are real numbers.

#### `extended-linear-srgb`

* Transfer function: identity

* Connection matrix: matrix to convert the [sRGB primaries](https://www.w3.org/TR/css-color-4/#valdef-color-srgb) to the primary colours specified in Rec. ITU-R BT.2100

#### `extended-srgb`

* Transfer function: See [extended-linear-srgb](#extended-linear-srgb)

* Connection matrix: matrix to convert the [sRGB primaries](https://www.w3.org/TR/css-color-4/#valdef-color-srgb) to the primary colours specified in Rec. ITU-R BT.2100

_Note:_ The domain of this transformation function is all real values. Its domain is not restricted to the unit interval [0, 1].

Also see note in the Issues section at the bottom about whether this space should be distinct from the existing `'srgb'` space.

#### `rec2100-hlg`

* Transfer function: HLG Reference OETF specified at Rec. ITU-R BT.2100

_Note:_ The range of the function is [0, 1].

* Connection matrix: Identity

_Note:_ Converting from `rec2100-hlg` to any SDR color space will not result in clipping.

#### `rec2100-pq`

* Transfer function: `EOTF<sup>-1</sup>[F<sub>D</sub>/300]` where `EOTF<sup>-1</sup>` is the inverse of the Reference PQ EOTF specified at Rec. ITU-R BT.2100.

_Note:_ The factor of 300 is such that a display luminance of 300 cd/m<sup>2</sup> results in a linear color value of 1 in the connection color space.

_Note:_ The domain of `EOTF<sup>-1</sup>` is [0, 10000]

* Connection matrix: Identity

### Compositing the HDR `HTMLCanvasElement`

The compositing behavior of an `HTMLCanvasElement` may be specified using the `configureHighDynamicRange` method.

#### Default behavior

This section describes the default compositing behavior for canvas element.
This is the behavior that happens if `configureHighDynamicRange` is not called, or if is called specifying `'default'` as the mode.

In this mode, the `HTMLCanvasElement` will be composited exactly as an `<img>` or `<video>` element with a source in the canvas' color space would be composited.

This means that if the canvas' color space is `'rec2100-hlg'` or `'rec2100-pq'`, then the canvas will be composited using high dynamic range, where available.

Note that if the canvas' color space is `'extend-srgb-linear'` or `'extended-srgb'`, then the canvas will not be composited using high dynamic range. Pixel values outside of the [0, 1] interval will extend the displayed gamut beyond sRGB, but not the displayed luminance beyond the maximum SDR luminance.

Performing appropriate tone mapping is the responsibility of the browser, the operating system, and the display device.
If the `configureHighDynamicRange` method is called with `CanvasHighDynamicRangeOptions` that specify metadata and the canvas' color space is `'rec2100-pq'`, then this metadata will be interpreted during compositing in the same way that it would be interpreted if included in a source displayed via an `<img>` or `<video>` element.

#### Extended mode

If the canvas' color space is `'extend-srgb-linear'` or `'extended-srgb'`, then pixels values outside of the [0, 1] interval will extend the displayed luminance beyond the maximum SDR luminance.

It is guaranteed that SDR colors in this mode exactly match SDR colors in non-HDR content on the page.
For example, a canvas pixel value of `'(1,0,0)'` in `'extended-srgb'` is guaranteed to match the CSS color `'red'`.

Performing appropriate tone mapping is the responsibility of the browser, the operating system, and the display device.
If the `configureHighDynamicRange` method is called with `CanvasHighDynamicRangeOptions` that specify a maximum luminance that is greater than the display's luminance, then tone mapping will be applied to prevent clipping of luminance values below to the specified maximum luminance.
Note that the tone mapping algorithm may not alter any SDR color values (otherwise the SDR color matching guarantee would be violated).

#### Passthrough mode

If the canvas' color space is `'extended-srgb-linear'`, then the canvas will be passed to the display device with no additional processing.

This mode does not make any guarantees about color matching with SDR content.

No tone mapping will be applied by the browser, operating system, or display device.

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

### The `extended-linear-srgb` color space

#### WebGL using extended mode

In this example, a WebGL application enables and HDR default drawing buffer, and clears it to the pixel value `(1,1,1,1)`.

```javascript
    var canvas = document.getElementById('MyCanvas');
    canvas.configureHighDynamicRange({mode:'extended'});

    var gl = canvas.getContext('webgl2');
    gl.drawingBufferStorage(gl.RGBA16F, canvas.width, canvas.height);
    gl.colorSpace = 'extended-linear-srgb';
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

Note that this is identical to specifying the subtitle color in `'extended-linear-srgb'`.

```javascript
    context.fillStyle = 'color(extended-linear-srgb 0.265  0.265 0.265)';
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

### Should we bother with the `extended` color spaces

Perhaps we shouldn't have an explicit ```'extended-srgb'``` color space.
The alternative is to define ```'srgb'``` to be extended by default (along with ```'display-p3'```, and ```'a98-rgb'```, and all the rest, presumably).

This was discussed earlier, but I don't remember where where the discussion landed.

### HDR compositing independent of color space

In the existing API, there is no way to have a linear space working space for an HLG or PQ canvas

The fix for this that I propose is to allow a `'hlg'` and `'pq'` `CanvasHighDynamicRangeMode`.

In that case, the code to work in a linearized HLG space would be:

```javascript
    canvas.configureHighDynamicRange({mode:'hlg'});
    var context = canvas.getContext('2d',
        {colorSpace:'extended-linear-srgb', storageFormat:'float-16'});
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
