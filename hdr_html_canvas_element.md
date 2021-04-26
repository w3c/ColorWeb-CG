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

### Constraints

There exist the following constraints.

* The exact maximum luminance of the output display is not known and not knowable.
  * The [CSS Media Queries Level 5 Specification](https://www.w3.org/TR/mediaqueries-5/#valdef-media-dynamic-range-high) allows the application to query the ``'dynamic-range'``. The resulting values are ``'standard'`` and ``'high'``.
  * The value changes over time.
  * The exact value is a fingerprinting vector.
* The exact number of nits of SDR content is also not known and not knowable.
  * On macOS, it is always 100.
  * On Windows, it depends on a user slider setting.
  * The exact value is a fingerprinting vector (again).

Because these values are not known, it is not possible for the application provide quantities related to display light.

### Proposed solution overview

The solution that we propose is to:

* Introduce new color spaces and precisions that are useful for HDR.
* Clearly define invertible and context-independent transformations between these spaces.
* Introduce the ability use more than 8 bits per pixel for a canvas element.
* Introduce the ability for an `HTMLCanvasElement` to configure HDR.

## Proposal

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

    // Passes 'extended-linear-srgb' through to the display device with no
    // tone mapping applied and no color matching guarantees.
    'passthrough',
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
    'float-16", 
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

### Color spaces

Update `PredefinedColorSpace` to include the following new color spaces.

```idl
  partial enum PredefinedColorSpace {
    'extended-linear-srgb',
    'extended-srgb',
    'rec2100-hlg',
    'rec2100-pq',
  }
```

### Conversion between color spaces

All `PredefinedColorSpace` are defined by how they are converted to XYZD50 under relative colorimetric intent.

Converting from XYZD50 to a `PredefinedColorSpace` is done by performing the inverse of the conversion to XYZD50.

Color values in XYZD50 may assume any real value (including values less than zero and greater than one).

All color space conversions thus have the following properties:

* They are invertible.
  * Caveat, invertible up to precision and clamping limitations.
* They are context independent.
  * There is one and only one way to convert from one space to another, and it does not depend on the operation being performed.
  * There is no implicit perceptual conversion (including tone mapping).
  * We will note where explicit tone mapping would fit in.
* They are path independent.
  * Converting from space A to C is the same as converting from space A to B to C.

#### `extended-linear-srgb`

To convert `extended-linear-srgb` to XYZD50 under relative colorimetric intent, perform the following steps:

* Apply the matrix transformation to convert the [sRGB primaries](https://www.w3.org/TR/css-color-4/#valdef-color-srgb) to the XYZ primaries.
* Apply the matrix transformation to convert the [sRGB white point](https://www.w3.org/TR/css-color-4/#valdef-color-srgb) to the D50 white point.

Note that the domain of this transformation function is all real values. Its domain is not restricted to the unit interval [0, 1].

#### `extended-srgb`

To convert `extended-srgb` to XYZD50 under relative colorimetric intent, perform the following steps:

* Convert each color channel to linear space.
  * For each color channel value `x`, this means applying the following function:
    * If `x < -0.4045`, then return `-pow((-x + 0.055)/1.055, 2.4)`
    * Else if `x <= 0.4045` then return `x / 12.92`
    * Else return `pow((x + 0.055)/1.055, 2.4)`
* Apply the matrix transformation to convert the [sRGB primaries](https://www.w3.org/TR/css-color-4/#valdef-color-srgb) to the XYZ primaries.
* Apply the matrix transformation to convert the [sRGB white point](https://www.w3.org/TR/css-color-4/#valdef-color-srgb) to the D50 white point.

Note that the domain of this transformation function is all real values. Its domain is not restricted to the unit interval [0, 1].
Also see note in the Issues section at the bottom about whether this space should be distinct from the existing `'srgb'` space.

#### `rec2100-hlg`

To convert `rec2100-hlg` to XYZD50 under relative colorimetric intent, perform the following steps:

* Apply the HLG inverse OETF defined in Table 5 of [ITU-R 2100](https://www.itu.int/dms_pubrec/itu-r/rec/bt/R-REC-BT.2100-2-201807-I!!PDF-E.pdf).
  * Note that this step of the transformation function is defined only on the domain of [0, 1].
  * Pixel values outside of that domain are clamped to that domain.
* Apply the matrix transformation to convert the primaries specified Table 2 of [ITU-R 2100](https://www.itu.int/dms_pubrec/itu-r/rec/bt/R-REC-BT.2100-2-201807-I!!PDF-E.pdf) to the XYZ primaries.
* Apply the matrix transformation to convert the white point from the reference white specified Table 2 of [ITU-R 2100](https://www.itu.int/dms_pubrec/itu-r/rec/bt/R-REC-BT.2100-2-201807-I!!PDF-E.pdf) to the D50 white point.

If this is followed, then converting from  `rec2100-hlg` to any SDR color space will not result in any luminance clipping.
This is a desirable property.

#### `rec2100-pq`

To convert `rec2100-pq` to XYZD50 under relative colorimetric intent, perform the following steps:

* Apply the reference PQ EOTF defined in Table 4 of [ITU-R 2100](https://www.itu.int/dms_pubrec/itu-r/rec/bt/R-REC-BT.2100-2-201807-I!!PDF-E.pdf).
  * Note that this step of the transformation function is defined only on the domain of [0, 1].
  * Pixel values outside of that domain are clamped to that domain.
* Apply the matrix transformation to convert the primaries specified Table 2 of [ITU-R 2100](https://www.itu.int/dms_pubrec/itu-r/rec/bt/R-REC-BT.2100-2-201807-I!!PDF-E.pdf) to the XYZ primaries.
* Apply the matrix transformation to convert the white point from the reference white specified Table 2 of [ITU-R 2100](https://www.itu.int/dms_pubrec/itu-r/rec/bt/R-REC-BT.2100-2-201807-I!!PDF-E.pdf) to the D50 white point.

If this is followed, then converting from `rec2100-pq` to any SDR color space will result in an undesirably dark image, because 10,000 nits will map to diffuse white.
The alternative is to scale the result so that, say, 100 nits maps to diffuse white.
That will be better in that it will be less undesirably dark, but it will be worse in that it will introduce severe luminance clipping.
There is no way to win here.
The most we can hope for is to make the math easy.

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

## Example Applications

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

#### WebGL using passthrough mode

If this example were changed to specify the passthrough mode then there would no longer be a guarantee that the canvas would match the CSS color `'white'`.

```javascript
    var canvas = document.getElementById('MyCanvas');
    canvas.configureHighDynamicRange({mode:'passthrough'});
```

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

