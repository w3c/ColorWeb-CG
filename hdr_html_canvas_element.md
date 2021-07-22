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

* in the case where `extended-linear-srgb` or `extended-srgb` are used, tone mapping is performed by the web app, using display capabilities provided by the platform.

![Tone mapping scenarios](./tone-mapping-scenarios.png)

#### Suggested Tone mapping for HDR content on sRGB displays
Not these are only used to tonemap HDR images at the point of rendering for display when the display is known to be an sRGB display.  They are not used for conversion between colour spaces which is defined in section XXXX.

##### PQ signal

##### HLG signal

_Input:_ Full-range non-linear floating-point `rec2100-hlg` pixel with black at 0.0 and diffuse white at 0.75. Values may exist outside the range 0.0 to 1.0.
_Output:_ Full-range non-linear floating-point `srgb` pixel with black at 0.0 and diffuse white at 1.0. Values may exist outside the range 0.0 to 1.0.
_Process:_
  1. Linearize the HLG signal exploiting its backwards compatibility with SDR consumer displays
  2. Convert from ITU BT.2100 color space to SRGB color space
  3. Convert to SRGB using the SRGB Inverse EOTF

_Note 3_ This transform utilises the backwards compatibility of ITU-R BT.2100 HLG HDR with consumer electronic displays.

```python
    def tonemapREC2100HLGtoSRGBdisplay(R,G,B):
      systemGamma = 2.2
      (r1,g1,b1) = hlg_ootf(R,G,B,systemGamma)
      (r2,g2,b2) = matrixXYZtoSRGB(matrixBT2020toXYZ(r1,g1,b1))
      (r3,g3,b3) = srgb_inverse_eotf(r2,g2,b2)
      (r4,g4,b4) = clipper_0_1(r3,g3,b3)
      return (r4,g4,b4)
```

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
* White chromaticity: `(0.3127, 0.3290)`

_Note:_ The system colorimetry specified in Rec. ITU-R BT.2100 is identical to that specified in Rec. ITU-R BT.2020.

The conversion from color space A to color space B is performed according to the following steps:

* apply the inverse transfer function of color space A
* convert to the connection space by multiplying by the connection matrix of color space A
* convert to color space B by multiplying by by the inverse of the connection matrix of color space B
* apply any linear light scaling to correctly map the level of perceived diffuse white level in to color space B
* apply the transfer function of color space B

The domain and range of this conversion consist of all real values and the component signals in the connection color space are real numbers.

#### `extended-linear-srgb`

* Transfer function: identity

* Connection matrix: matrix to convert the [sRGB primaries](https://www.w3.org/TR/css-color-4/#valdef-color-srgb) to the primary colours specified in Rec. ITU-R BT.2100

* Linear light scaling: 1.0

#### `extended-srgb`

* Transfer function: See [extended-linear-srgb](#extended-linear-srgb)

* Connection matrix: matrix to convert the [sRGB primaries](https://www.w3.org/TR/css-color-4/#valdef-color-srgb) to the primary colours specified in Rec. ITU-R BT.2100

* Linear light scaling: 1.0

_Note:_ The domain of this transformation function is all real values. Its domain is not restricted to the unit interval [0, 1].

Also see note in the Issues section at the bottom about whether this space should be distinct from the existing `'srgb'` space.

#### `rec2100-hlg`

* Transfer function: HLG Reference EOTF specified at Rec. ITU-R BT.2100 with a System Gamma of Unity

* Connection matrix: Identity

* Linear light scaling: 1.0/0.265

_Note:_ Converting from `rec2100-hlg` to any SDR color space will not result in clipping.

#### `rec2100-pq`

* Transfer function: `EOTF<sup>-1</sup>[F<sub>D</sub>/300]` where `EOTF<sup>-1</sup>` is the inverse of the Reference PQ EOTF specified at Rec. ITU-R BT.2100.

* Connection matrix: Identity

* Linear light scaling: 1.0/0.265

_Note:_ The factor of 300 is such that a display luminance of 300 cd/m<sup>2</sup> results in a linear color value of 1 in the connection color space.

_Note:_ The domain of `EOTF<sup>-1</sup>` is [0, 10000]


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

```python
    def convertExtendedSRGBtoREC2100HLG(R,G,B):
      systemGamma = 1.0
      linearLightScaler = 0.265
      r1 = srgb_eotf(R)
      g1 = srgb_eotf(G)
      b1 = srgb_eotf(B)
      (r2,g2,b2) = matrixXYZtoBT2020(matrixSRGBtoXYZ(r1,g1,b1))
      r3 = linearLightScaler * r2
      g3 = linearLightScaler * g2
      b3 = linearLightScaler * b2
      (r4,g4,b4) = hlg_inverse_ootf(r3,g3,b3,systemGamma)
      (r5,g5,b5) = hlg_oetf(r4,g4,b4)
      return (r5,g5,b5)
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

```python
    def convertExtendedLinearSRGBtoREC2100HLG(R,G,B):
      systemGamma = 1.0
      linearLightScaler = 0.265
      (r2,g2,b2) = matrixXYZtoBT2020(matrixSRGBtoXYZ(r1,g1,b1))
      r3 = linearLightScaler * r2
      g3 = linearLightScaler * g2
      b3 = linearLightScaler * b2
      (r4,g4,b4) = hlg_inverse_ootf(r3,g3,b3,systemGamma)
      (r5,g5,b5) = hlg_oetf(r4,g4,b4)
      return (r5,g5,b5)
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

  ```python
      def convertREC2100HLGtoExtendedSRGB(R,G,B):
        systemGamma = 1.0
        linearLightScaler = 1.0 / 0.265
        (r1,g1,b1) = hlg_inverse_oetf(R,G,B)
        (r2,g2,b2) = hlg_ootf(r1,g1,b1,systemGamma)
        r3 = linearLightScaler * r2
        g3 = linearLightScaler * g2
        b3 = linearLightScaler * b2
        (r4,g4,b4) = matrixXYZtoSRGB(matrixBT2020toXYZ(r3,g3,b3))
        (r5,g5,b5) = srgb_inverse_eotf(r4,g4,b4)
        return (r5,g5,b5)
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

    ```python
        def convertREC2100HLGtoExtendedSRGB(R,G,B):
          systemGamma = 1.0
          linearLightScaler = 1.0 / 0.265
          (r1,g1,b1) = hlg_inverse_oetf(R,G,B)
          (r2,g2,b2) = hlg_ootf(r1,g1,b1,systemGamma)
          r3 = linearLightScaler * r2
          g3 = linearLightScaler * g2
          b3 = linearLightScaler * b2
          (r4,g4,b4) = matrixXYZtoSRGB(matrixBT2020toXYZ(r3,g3,b3))
          return (r4,g4,b4)
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
