# Canvas High Dynamic Range

## Proposal summary / TLDR

This summary will use terminology from the definitions section below.

### Use cases

There three main classes of HDR use cases that inform this proposal. They are:

* To draw content that can precisely color-match existing SDR content, while allowing to take advantage of HDR headroom.
  * E.g, adding HDR to an existing SDR application.
  * E.g, using SDR HTML UI along with an HDR application, with guaranteed color matching.
* To display PQ encoded HDR images that are drawn to a canvas
  * E.g, displaying PQ content.
  * E.g, working in physical luminance.
* To display HLG encoded HDR images that are drawn to a canvas
  * E.g, displaying HLG content.
  * E.g, using a fixed signal range that maps to the full display device luminance range.

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

### Proposed solution

The solution that we propose is to:

* Introduce the term device independent light.
* Introduce new color spaces and precisions that are useful for HDR.
  * These color spaces are best interpreted as signals, which have a well defined conversion into device independent light.
* Introduce an HTMLCanvasElement method through which an HDR compositing mode may be specified.
  * The compositing mode corresponds to a class of opto-optical transfer functions that are applied to the device independent light defined by the canvas' buffer's contents to determine the canvas' final display light.
  * We introduce three HDR compositing modes, matching the three main use cases.
  * Note that these are modes of compositing a canvas, are independent of the canvas' color space, and may be changed between dynamically.
* Define mappings from SDR, PQ, and HLG signals into display independent light.
  * Mappings are chosen to complement the HDR compositing modes.
  * Remaining free parameters in the HLG mapping are chosen to ensure a smooth fallback when HDR is disabled.
  * Remaining free parameters in the PQ mapping are chosen to make math easier.
* If the application can overcome the fingerprinting limitations (e.g, by just asking the user), any desired behavior can be accomplished, using appropriate math.

## Definitions

In this section we provide definitions for the terminology that is used throughout the document.
These definitions focus on being precise about brightness, and will not be as precise about color.

### Display light, scene light, device independent light, and signal

#### Display light

Display light is the light that is emitted by a display device (e.g, a television) at each pixel.
Display light has the property that it is limited by a maximum and minimum value that the display device can produce.

Display light is a precise physical quantity, namely, luminance in nits (nits are cd/m^2).

Display light is sometimes treated as being in the interval [0, 1], with 1 representing the maximum luminance the device can produce, and 0 possibly representing the minimum luminance that the device can produce.
In these cases display light is sometimes referred to as being "linear", because nothing more than a linear (or more precisely, affine) transformation has been applied to it.
We will not use this convention.

#### Relative scene linear light

Relative scene linear light is linearly proportional to the physically measurable light in a scene being captured.
The precise scaling factor used in mapping from physically measurable light to relative scene linear light depends on the particular capture setup, including such variables as camera exposure.
The scaling of relative scene linear light is such that values of scene linear light are limited to the interval [0, 1].

This document uses the term "scene light" as shorthand for "relative scene linear light".
TODO: Fix this if it is a source of confusion.

#### Device independent light

Device independent light is a space into which all content (SDR, PQ, and HLG) has a well-defined and simple mapping.

The reason for the existence of device independent light is that there needs to exist a space that does not depend on the display device, in which content is clearly defined.

Device independent light is extremely similar to relative scene linear light.
Earier versions of this document used the term scene light (meaning relative scene linear light) instead of device independent light, but this was too confusing because of the issues of artistic adjustments and the undefined nature of scene light for SDR content.

#### Signal

A signal is a digial encoding of light.

An example of a signal is the pixel values in an sRGB image.
Other examples include the pixel values of PQ or HLG images.

Most signals take values in the [0, 1] interval and are represented using fixed-point (e.g, the usual 8-bit encoding of an sRGB image).
In this document, we will be allowing for signals that take any real value, and are represented using floating-point.

### Transfer functions

#### Opto-electronic transfer function (OETF)

An opto-electronic transfer function (OETF) is a function that transforms scene light (or sometimes modified scene light) to a signal.

An example of an OETF is the HLG OETF, specified in the second row of Table 5 of the [BT.2100 specification](https://www.itu.int/dms_pubrec/itu-r/rec/bt/R-REC-BT.2100-2-201807-I!!PDF-E.pdf).

Another example of an OETF is the PQ OETF, specified in the final row of Table 4 of the [BT.2100 specification](https://www.itu.int/dms_pubrec/itu-r/rec/bt/R-REC-BT.2100-2-201807-I!!PDF-E.pdf).

A non-example of an OETF is the [sRGB function](https://www.w3.org/Graphics/Color/srgb) ``f(x)=(x<0.0031308) ? 12.92*x : 1.055*pow(x,1/2.4)-0.055``.
The domain of this OETF-like thing is the interval [0, 1], with diffuse white at 1.
The range (signal) is standard sRGB pixel values in the interval [0, 1].
This is not an OETF because sRGB only specifies display light for sRGB signals, it does not define a relationship to scene light (in other words, the domain of this function is not scene light).

#### Opto-optical transfer function (OOTF)

An opto-optical transfer function (OOTF) is a function that transforms scene light to display light.

It is the responsibility of the OOTF to reproduce the appearance of the scene, within the capabilities of the display device, and subject to viewer's environment.

The HLG OOTF is parameterized by the maximum luminance of the display device. It maps a fixed range of scene light to cover the full range (up to maximum luminance) of the display device.

The reference PQ OOTF is close to the identity function, with scene light almost equaling display light.
In practice (that is, when applied to a non-reference display), the PQ OOTF is parameterized by content metadata and the capabilities of the display device.

#### Electro-optical transfer function (EOTF)

An electro-optical transfer function (EOTF) is a function that transforms signal to display light, and can be computed as the OOTF applied to the inverse-OETF applied to a signal.
We are including this definition for completeness, but this document will be written in terms of OETFs and OOTFs.

#### Device independent electro-optical transfer function (DI-EOTF)

A device independent electro-optical transfer function (DI-EOTF) is a function that transforms a signal to display independent light.

In this proposal we will define the DI-EOTF for sRGB, PQ, and HLG signals.

#### Device independent opto-optical transfer function (DI-OOTF)

A device independent opto-optical transfer function (DI-OOTF) is a function that transforms display independent light to display light.

In this proposal we will define three classes of DI-OOTFs that the application may use to display HDR content.

### Application versus browser

In this document, the term application will refer to the web application that does not have direct access to the operating system.
This is in contrast with the browser, which is a native application with direct access to the operating system.

## New color spaces and storage formats

The existing CanvasColorSpaceProposal has been narrowed down to supporting just ``'srgb'`` and ``'display-p3'``.
It no longer covers adding other color spaces, or changing buffer formats.

### Color spaces

The pixels values in a canvas back buffer are in a specific color space.
The default color space is sRGB.

This new proposal expands the set of supported color spaces to include:

* ``'srgb'``
* ``'display-p3'``
* ``'srgb-linear'``
* ``'display-p3-linear'``
* ``'rec2020-linear'``

#### Interpretation in terms of signal

The canvas back buffer is best understood as a signal, or an encoded version of device independent light.

The color space determines this encoding.
In particular, the color space specifies a DI-EOTF that maps the signal in from device independent light.
For all of the above-listed color spaces, this DI-EOTF and its inverse are defined for all real numbers.

The pixel value of (1,1,1) in all color spaces corresponds diffuse white.

(The reader may note that color primaries are not being attended to in this section. Their neglect will continue.)

#### Relationship to BT.2100 floating-point signal representation.

When a canvas has color space ``'rec2020-linear'``, the back buffer's pixel values are precisely the scene-referred signal referred to in Table 10 of the [BT.2100 specification](https://www.itu.int/dms_pubrec/itu-r/rec/bt/R-REC-BT.2100-2-201807-I!!PDF-E.pdf).

### Storage formats

This proposal includes adding 16-bit floating-point as a supported storage format for 2D Canvas.

ImageData will also be updated to support 16-bit fixed-point and 32-bit floating-point.

## HDR compositing modes

In this section we go over the three HDR compositing modes that we propose to make available.
These are compositing modes, which means that their effects are visible to the user via the display device, but are not visible to the application itself.

As discussed in the previous section, the contents of a canvas' buffer may be interpreted as a signal that has a well-defined mapping into display independent light.
The compositing mode selects which class of device independent opto-optical transfer function is to be applied to that display independent light to convert it to display light produced by the display device.
It is the responsibility of the browser, the operating system, the graphics hardware, and the display hardware to effect the application-selected OOTF.

In this section, we will define how we will map PQ and HLG signals to our set of defined color spaces.
These are necessarily raw signal-to-signal mappings, done without knowledge of the display device or viewing environment.

### Mode 1: Extended-SDR mode or SDR-relative luminance (for matching SDR colors)

The requirement of this mode is that all SDR colors inside the HDR Canvas match exactly their appearance in an SDR canvas.

In this mode, color values outside of the range of [0, 1] may be used to represent luminance beyond the SDR range.
The exact luminance of such a color value is expressed relative to the maximum SDR luminance, rather than in absolute nits.
For example, the color ``color(srgb-linear 2 2 2)`` is exactly twice as bright as ``color(srgb 1 1 1)``, but it not known how many nits that is.

#### Metadata and tone mapping

There is no limit on the maximum luminance that can be expressed by a pixel value in this mode.
If no additional metadata is provided, then all pixels will be clamped to the display device's maximum luminance.

To avoid aggressive clamping, metadata may be provided.
This metadata consists of the scene's maximum luminance, as a multiple of the SDR luminance.
A combination of the browser, operating system, and display device are responsible for mapping the canvas' content into the display device's capabilities.

Note that this mapping has the very severe restriction that it must not alter SDR colors found inside the HDR canvas (in other words, the mapping must be the identity on all SDR values).

Also note that no mastering primaries or white point are specified, because there is no way to incorporate that data into a mapping that remains the identity on all SDR values.

#### Remarks on operating system interaction

On Windows, this mode opts the canvas in to being affected the SDR slider (like any non-HDR canvas would be).

### Mode 2: Physical luminance (for displaying PQ content)

The goal of this mode is to allow faithful display of PQ encoded content.
A PQ image drawn to a canvas that is composited in this mode will appear the same as that image when drawn by an ``img`` element on the page.

That is not the only use of this mode.
This mode will also allow the application to specify physical luminance values to be displayed (to the extent that they are respected by the operating system and display device).

#### Converting between PQ signals and display independent light

When drawing a PQ image to a canvas and then displaying that canvas, there are two independent operations being performed.
* The input PQ image must be transformed into device independent light to be encoded into the canvas' signal (or canvas' pixel values).
* The output canvas' signal must be transformed into device independent light that is then transformed into display light

The goal of this mode is that when these two operations are performed in sequence on a PQ image, the resulting transformation is the identity (this is ignoring metadata for the moment).

We are forced to choose a mapping between PQ signals (which have a precise meaning in nits) and device independent light (to which we have assiduously avoided assigning a precise meaning in nits so far).
The mapping we choose is that the device independent light value of 1 (that of ``color(srgb 1 1 1)``) map to the PQ signal value that represents 100 nits.

The motivation for 100 nits is that:
* It's a reasonable value with precedent.
* It makes the math easier for anyone who desires a different value (and many applications will want many other values).
* It allows scene light be interpreted as being in hecto-nits, if one wishes to coerce such an interpretation.
* This maps to the behavior of ``kCGColorSpaceExtendedSRGB`` and other CoreGraphics color spaces.

If one wants to draw a color that will appear as 203 nits, this can be done with ``color(srgb-linear 2.03 2.03 2.03)``.
Similar math may be done to determine values to write to the WebGL or WebGPU swap chain.

#### Alternative mapping choices: 80 nits

There exists a reasonable argument for mapping 1 unit in device independent light to 80 nits.
This is the reference display white point luminance according to [IEC 61966-2-1](http://www.color.org/chardata/rgb/srgb04.xalter).
It is also the number that is used by ``DXGI_COLOR_SPACE_RGB_FULL_G10_NONE_P709`` on Windows.

#### Alternative mapping choices: 203 nits

[ITU 2408-3](https://www.itu.int/dms_pub/itu-r/opb/rep/R-REP-BT.2408-3-2019-PDF-E.pdf) recommends mapping 1 unit in device independent light to 203 nits.
This matches the behavior of no efficient paths in any operating systems, and is not a round number.

#### Device independent opto-optical transfer function and metadata

The default DI-OOTF of this mode (if no metadata is provided) is a multiplication by 100 (converting the device independent light value of 1 to display light value of 100 nits), and then clamping to the display device's maximum luminance.

To avoid aggressive clamping, the usual complement of HDR10 metadata may be provided (min luminance, max luminance, primaries, white point, CLL, and ALL).
As in the previous mode, a combination of the browser, operating system, and display device are responsible for providing an appropriate DI-OOTF that takes and the display device's capabilities into account.

Note that unlike the previous mode, SDR colors are not sacrosanct in this mode, and SDR colors may be aggressively transformed.

#### Remarks on operating system interaction

On Windows, this mode opts the canvas out of being affected by the SDR slider.

On macOS, ``color(srgb 1 1 1)`` is always treated as matching PQ's 100 nits.
The actual number of nits displayed may vary widely depending on ambient lighting and display device capabilities (reference modes to disable these adaptations are available in operating system settings).
Consequently, if no metadata is specified (and therefore tone mapping is disabled), then this mode is equivalent to the previous mode.

### Mode 3: HLG transformation of device independent light

The goal of this mode is to allow faithful display of HLG encoded content.
An HLG image drawn to a canvas that is composited in this mode will appear the same as that image when drawn by an ``img`` element on the page.

That is not the only use of this mode.
An application that wishes to produce pixel values in the interval [0, 1] in device independent light space and have that interval gracefully map to the full display's luminance range (be that range HDR or SDR) would find this mode a natural fit.

#### Background on converting HLG signals to display light

Before discussing what we propose to do with canvas elements, we will first describe the mechanism for displaying an HLG image on a display that uses raw luminance values as its signal, and has a specified maximum luminance.
A more complete treatment of this topic may be found in the very readable [PQ to HLG Transcoding](https://www.bbc.co.uk/rd/sites/50335ff370b5c262af000004/assets/592eea8006d63e5e5200f90d/BBC_HDRTV_PQ_HLG_Transcode_v2.pdf) or in the [BT.2100 specification](https://www.itu.int/dms_pubrec/itu-r/rec/bt/R-REC-BT.2100-2-201807-I!!PDF-E.pdf).

Transforming from HLG signal to raw luminance has three steps.

The first step is to convert from the HLG signal into scene light. This is done by the inverse-OETF function.
The function's domain is [0, 1], and its range is normalized to [0, 1], with 0.5 mapping to 1/12.
(Sometimes this range is normalized to [0, 12], but we do not use that convention).
This step is independent of the output display.

The second step is to apply the normalized OOTF to transform scene light into normalized display light.
This function is a gamma ramp, parameterized by the maximum luminance of the display, with a domain and range of [0, 1].

The third step is to scale that value in [0, 1] by the display device's maximum luminance (in nits) to arrive at the physical display light quantity.

#### Device independent opto-optical transfer function

The DI-OOTF of this mode is the HLG OOTF, which will map the interval [0, 1] in device independent light to the display device's full luminance range.

In this mode, the color ``color(srgb 1 1 1)`` represents maximum luminance.
All higher luminance values are outside of the domain of the HLG OOTF, and are clamped.

#### Remarks on choice of device independent light space mapping

For displaying HLG content, there are three conventions for the interval to be used for scene light space.

* [0, 1], which we will call normalized scene light space.
* [0, 12], which we will call non-normalized scene light space.
* The [BT.2408 Recommendation](https://www.itu.int/dms_pub/itu-r/opb/rep/R-REP-BT.2408-3-2019-PDF-E.pdf) is that an HLG signal value of 0.75 map to diffuse white, which we have previously defined to be the scene light value of 1.
  * If we are to apply no OOTFs, then this would mean scaling the interval to be [0, 1/0.265], because 0.265 is the pre-image of 0.75 under the HLG OETF.
  * If we were to bend the definition of scene light and apply the semi-relevant 1000-nit-max OOTF, then the interval would be [0, 1/0.203].

We select one of these for our definition of device independent light.

The third choice is to be discarded because it is a descent into chaos.
If an application desires to specify a CSS style that coincides with an HLG signal of 0.75, the application is free to do a bit of math and specify the pre-image of the HLG signal of 0.75, previously noted above to be ``color(srgb-linear 0.265, 0.265, 0.265)``.

There are two good reasons to prefer normalized scene light to non-normalized scene light.

The first reason to prefer normalized scene light is that the numbers are round and predictable.

The second reason to prefer normalized scene light is that the non-normalized representation will not behave well if applied to a non-SDR canvas. In particular, it will saturate and start to clamp at a signal value of 0.5.
A defining property of HLG is the smooth transition between SDR and HDR, this clamping is not a smooth transition, and so something different would need to be done in non-SDR canvases.
That would end up with a different scene light representation of an HLG image depending on whether it is destined for an HDR or SDR canvas, which is a mess from the perspective of the application and the browser.

### Mode 0: SDR

For completeness, there does exist one more HDR mode, the no-HDR HDR mode (which is the current default behavior).

In this mode, device independent light is clamped to the range [0, 1].
This is done by having the native application (the browser in this case) provide the operating system with a buffer in a specified color space.
The EOTF is then applied by some combination of the operating system, graphics hardware, and display device.

For HLG images, the behavior that falls out is for the OOTF to not be applied.
The HLG OOTF is the identity function if the display maximum luminance is 334 nits, so, for HLG images, the behavior for non-HDR canvases will be to display them as though the target display were 334 nits, which is a reasonable maximum luminance value to assume an SDR monitor has.

For PQ images, the behavior that falls out is for the images to be clamped beyond 100 nits.

## Proposed API outline

To enable the configuration of HDR compositing.

```idl
  partial interface HTMLCanvasElement {
    void configureHDR(HTMLCanvasCompositingMode mode, optional HTMLCanvasHDRMetadata metadata);
  }

  enum HTMLCanvasHDRCompositingMode {
    'disabled',
    'extended', // Call this "relative-luminance"?
    'pq-compatible', // Call this "absolute-luminance" or "physical-luminance"?
    'hybrid-log-gamma',
  }

  // TODO: This feels sloppy to have all HDR parameters be in a single
  // dictionary.
  dictionary HTMLCanvasHDRMetadata {
    // Value specified in multiples of SDR white.
    // Used by 'extended' mode.
    float? maxExtendedRangeValue;

    // Values specified in nits.
    // Used by 'pq-compatible' mode.
    float? maxPhysicalLuminance;
    float? minPhysicalLuminance;

    // Values specified in CIE1931.
    // Used by 'pq-compatible' mode.
    float? redPrimaryX;
    float? redPrimaryY;
    float? greenPrimaryX;
    float? greenPrimaryY;
    float? bluePrimaryX;
    float? bluePrimaryY;
    float? whitePointX;
    float? whitePointY;
  }
```

To enable 2D Canvas to use more than 8 bits per pixel.

```idl
  enum CanvasStorageFormat {
    "unorm8",
    "float16", 
  }

  dictionary CanvasRenderingContext2DSettings {
    CanvasStorageFormat storageFormat = "unorm8";
  }
```

To expose additional color spaces.

```idl
  partial enum CanvasColorSpace {
    "srgb-linear",
    "display-p3-linear",
    "rec2020-linear",
  }
```

To enable ImageData to use more than 8 bits per pixel.

```idl
  enum ImageDataStorageFormat {
    "unorm8",
    "unorm16", 
    "float32", 
  }

  typedef (Uint8ClampedArray or Uint16Array or Float32Array) ImageDataArray;
  partial dictionary ImageDataSettings {
    ImageDataStorageFormat storageFormat = "unorm8";
    ImageDataArray data;
  }
```

## Example Applications

### Example Set 1: Application With HTML UI on top of a Canvas

In this example set we consider an application in which an HTML UI is used with content in an HTML Canvas.

The beginning of these examples is the same.
We have a canvas, and a green HTML element on top of it.

```xml
<html>
<body>
<div style='position: relative; width: 500px; height: 500px;'>
  <canvas id='MyCanvas' width=500px height=500px
          style='position:absolute; width: 100%; height:100%;'></canvas>
  <div id='MyGreenUI' style='position:absolute; left:10px; top:10px;
                             width:100px; height:20px;
                             background-color:rgb(0, 255, 0);'>
    MyUIText
  </div>
</div>  
<script>
  var canvas = document.getElementById('MyCanvas');
  // ... examples diverge here ...
</script>
</body>
</html>
```

We can consider two different types of applications in this general context.

#### Example 1A: Application wants the HTML UI to color-match with the HDR Canvas

In this example, the application wants color matching with the HTML UI.
The most concrete example of this would be a situation where the application uses the HTML color picker element to select colors to use inside the canvas.

In this case, the application will want to use the default mode of ``'extended'``.

```javascript
    canvas.configureHDR('extended');

    // This green will match the color in MyGreenUI.
    var context = canvas.getContext('2d', { precision: 'float16' });
    context.fillStyle = 'rgb(0, 255, 0)';
    context.fillRect(50, 50, 20, 20);
```

#### Example 1B: Same as 1A, but using WebGL

This is the same as the above example, but the application is using WebGL.

```javascript
    canvas.configureHDR('extended');

    // This green will match the color in MyGreenUI.
    var gl = canvas.getContext('webgl2');
    gl.drawingBufferStorage(gl.RGBA16F, canvas.width, canvas.height);
    gl.colorSpace = 'srgb-linear';
    gl.clearColor(0.0, 1.0, 0.0, 1.0);
    gl.clear(gl.COLOR_BUFFER_BIT);
```

#### Example 1C: Application wants the HTML UI to be SDR content relative to the HDR Canvas

In this example, the application does not want color matching with the HTML UI.
The application views the HTML UI as being regular SDR content that is sitting on top of an HDR canvas.

* The UI should be displayed however the underlying OS displays SDR content.
* The HDR canvas should be displayed however the underlying OS displays HDR content.

The application in this case will want ``'pq-compatible'``.

```javascript
    canvas.configureHDR('pq-compatible');

    // This green will LIKELY NOT match the color in MyGreenUI.
    var context = canvas.getContext('2d', { precision: 'float16' });
    context.fillStyle = 'rgb(0, 255, 0)';
    context.fillRect(50, 50, 20, 20);
```

Note the phrasing of "likely not".
On Windows, if the SDR slider is set "just so", then they may happen to match.
On macOS, if there is no tonemapping applied to the canvas, then they will happen to match.

### Example Set 2: Using HDR for effects inside a Canvas element

In this set of examples, we have an existing SDR application.
This application wishes to add HDR for UI effects or for special effects.

In these examples, the application will want ``'extended'``, because that mode is guaranteed not to be disruptive to the application as it already exists.

#### Example 2A: Brightening part of a UI to draw user's attention

In this example, there exists a helpful piece of UI that the application wants to draw the user's attention to.
That UI is being drawn by the canvas element.

To draw the user's attention, the application pulses a doubling of the brightness of that part of the UI code.

```xml
  <html>
  <body>
  <canvas id='MyCanvas' width=500px height=500px
          style='position:absolute; width: 100%; height:100%;'></canvas>
  <script>
    var canvas = document.getElementById('MyCanvas');
    canvas.configureHDR('extended');
    var context = canvas.getContext('2d', { precision: 'float16' });

    function animateBrightenedUI(timeInSections) {
      // This code will brighten the content up to a factor of 2.
      var brighteningFactor = 1 + Math.abs(Math.sin(timeInSeconds * 2*Math.PI));
      context.filter = 'brightness(' + brighteningFactor + ')';

      // And this code draws the regular SDR UI.
      context.fillStyle = 'white';
      context.fillRect(100, 100, 120, 20);
      context.fillStyle = 'black';
      context.fillText('I Am Some Helpful Text', 105, 115);
    }
  </script>
  </body>
  </html>
```

#### Example 2B: An effect in a WebGL application

Consider an existing WebGL application with a lens flare effect.
The lens flare is currently an 8-bit texture, but the application wants to have it actually be HDR, without changing any of the rest of application.

Here is an outline of the previously existing application code. An existing HTMLCanvasElement, ``canvas`` is assumed to exist.

```javascript
    var gl = canvas.getContext('webgl2');

    var myLensFlareData = new Uint8ClampedArray(4 * 256 * 256);
    // Populate the data.

    const tex = gl.createTexture();
    gl.bindTexture(gl.TEXTURE_2D, tex);
    gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA8, 256, 256, 0, gl.RGBA,
                  gl.UNSIGNED_BYTE, myLensFlareData);
    gl.useProgram(myLensFlareProgram);
    gl.drawArrays(primitiveType, offset, count);
  }
```

It is possible to enable just this one HDR texture, with minimal changes to the existing application.
The comments in this code show the changes that are needed.

```javascript
    // Enable HDR for the canvas. Use the default mode of 'extended',
    // because that will match the existing application behavior.
    canvas.configureHDR('extended');

    var gl = canvas.getContext('webgl2');

    // Configure the swap chain to be floating-point. Leave the color space as
    // the default of 'srgb'. This will leave all colors in [0, 1] unchanged,
    // but allow for specifying colors outside of the range of [0, 1].
    gl.drawingBufferStorage(gl.RGBA16F, canvas.width, canvas.height);
    gl.colorSpace = 'srgb-linear';

    // This time our data is a Float32Array, and we'll be writing values in the
    // extended sRGB color space, including values outside of [0, 1].
    var myLensFlareDataInExtendedSRGB = new Float32Array(4 * width * height);
    myLensFlareDataInExtendedSRGB[...] = ...

    const tex = gl.createTexture();
    gl.bindTexture(gl.TEXTURE_2D, tex);

    // Change the texture we're sampling from to be RGBA16F.
    gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA16F, 256, 256, 0, gl.RGBA,
                  gl.FLOAT, myLensFlareDataInExtendedSRGB);

    // Draw using the same program as before. It will sample the texture into
    // floating-point variables, and write them to the gl_FragColor.
    gl.useProgram(myLensFlareProgram);
    gl.drawArrays(primitiveType, offset, count);
  }
```

### Example Set 3: Displaying a PQ image inside a Canvas element

In this set of examples we assume the existence of a PQ-encoded image.
Because the image is encoded in PQ, which is physical luminance, we will want the canvas to be interpreted as being in physical luminance.

#### Example 3A: Displaying a PQ image in a 2D Canvas as intended

In this example, we use a 2D Canvas to display a PQ image.
The nits specified by PQ image will match the nits displayed on the screen, as much as is allowed by the underlying operating system and the physical display.

```xml
  <html>
  <body>
  <canvas id='MyCanvas' width=2048px height=858px
          style='position:absolute; width: 95%;'></canvas>
  <script>
    var canvas = document.getElementById('MyCanvas');
    canvas.configureHDR('pq-compatible');
    var context = canvas.getContext('2d', { precision: 'float16' });

    var image = new Image();
    image.onload = function() {
      context.drawImage(image, 0, 0, 2048, 858);
    }
    var url = 'https://storage.googleapis.com/dalecurtis/cosmos_1000_pq_hdr.avif';
    image.src = url;
  </script>
  </body>
  </html>
```

#### Example 3B: Displaying a PQ image in a 2D Canvas NOT as intended

In this example, we make a mistake, and use ``'extended'`` in the above exmple.

```javascript
  canvas.configureHDR('extended');
```

What happens?

On Windows, the result will be that the image will be affected by the SDR slider.
This means that it will likely be brighter than is intended by the content author.

On macOS, the result will likely be indistinguishable.

#### Example 3C: Displaying a PQ image with a subtitle

This example builds on example 3A, but adds a subtitle text to the image.

First consider the following code, where the subtitle specifies its color as being sRGB white.
In this case, the subtitle will appear as 100 nits.

```javascript
  image.onload = function() {
    context.drawImage(image, 0, 0, 2048, 858);
    context.font = "128px Arial";
    context.fillStyle = 'white';
    context.fillText('Hello, I am a subtitle!', 400, 800);
  }
```

#### Example 3D: Displaying a PQ image with a 300 nit subtitle using CSS Color Level 4

Suppose the application does not want 100 nit subtitles, but would prefer 300 nit subtitles.
The easiest way to specify this would be to use CSS Color Level 4 syntax, in the ``srgb-linear`` color space.
In that space, the color values may be interpreted as hundreds of nits (or hectonits).
In that way, 300 nits would be represented by the style ``'color(srgb-linear 3 3 3)'`` as follows.

```javascript
    context.font = "128px Arial";
    context.fillStyle = 'color(srgb-linear 3 3 3)';
    context.fillText('Hello, I am a subtitle!', 400, 800);
```

#### Example 3E: Displaying a PQ image with an 80 nit subtitle using a brightness filter

Alternatively, the application could use a brightness filter to achieve a different nit level for SDR content.
In this example, the application selects 80 nits for its subtitle.

```javascript
  image.onload = function() {
    context.drawImage(image, 0, 0, 2048, 858);
    context.filter = 'brightness(0.8)';
    context.font = "128px Arial";
    context.fillStyle = 'white';
    context.fillText('Hello, I am a subtitle!', 400, 800);
  }
```

### Example Set 4: Displaying an HLG image inside a Canvas element

In this set of examples we assume the existence of a HLG-encoded image.
We expect this HLG-encoded image to be displayed in a way that takes advantage of the full luminance of the display device.

#### Example 4A: Displaying an HLG image in a 2D Canvas as intended

In this example, we use a 2D Canvas to display a HLG image.
We set the canvas to use the appropriate compositing mode.

```xml
  <html>
  <body>
  <canvas id='MyCanvas' width=2048px height=858px
          style='position:absolute; width: 95%;'></canvas>
  <script>
    var canvas = document.getElementById('MyCanvas');
    canvas.configureHDR('hybrid-log-gamma');
    var context = canvas.getContext('2d', { precision: 'float16' });

    var image = new Image();
    image.onload = function() {
      context.drawImage(image, 0, 0, 2048, 858);
    }
    var url = 'https://storage.googleapis.com/dalecurtis/cosmos_hlg.avif';
    image.src = url;
  </script>
  </body>
  </html>
```

#### Example 4B: Displaying an HLG image in a 2D Canvas NOT as intended

In this example, we make a mistake, and use ``'extended'`` in the above exmple.

```javascript
  canvas.configureHDR('extended');
```

What happens?

The HLG image will be limited to the SDR range (and will be drawn as though on a 334-nit SDR device).

#### Example 4C: Displaying an HLG image with a subtitle

This example builds on example 3A, but adds a subtitle text to the image.

First consider the following code, where the subtitle specifies its color as being sRGB white.
In this case, the subtitle will appear as maxmum brightness.

```javascript
  image.onload = function() {
    context.drawImage(image, 0, 0, 2048, 858);
    context.font = "128px Arial";
    context.fillStyle = 'white';
    context.fillText('Hello, I am a subtitle!', 400, 800);
  }
```

It is a convention in HLG to use a signal value of 0.75 as diffuse white.
To accomplish this, we would convert HLG's 0.75 to a CSS color value.

```javascript
  context.fillStyle = 'color(srgb-linear 0.265  0.265 0.265)';
```

