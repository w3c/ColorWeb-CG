# Canvas High Dynamic Range

## Proposal summary / TLDR

This summary will use terminology from the definitions section below.

### Use cases

There three main classes of HDR use cases that inform this proposal. They are:

* To draw HDR content with the minimum performance overhead.
* To display HLG encoded HDR images and video in a canvas.
  * Such that these images appear exactly the same as they would if displayed via an```<img>``` or ```<video>``` tag.
* To display PQ encoded HDR images and video in a canvas.
  * Such that these images appear exactly the same as they would if displayed via an ```<img>``` or ```<video>``` tag.

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

* Introduce the ability for an ```HTMLCanvasElement``` to enable HDR.
* Introduce the ability use more than 8 bits per pixel for a canvas element.
* Introduce new color spaces and precisions that are useful for HDR.
* Clearly define invertible and context-independent transformations between these spaces.

## Proposal

### Enabling HDR on a canvas element

Add a new ```HTMLCanvasElementHighDynamicRangeOptions``` dictionary with HDR configuration options.

```idl
  dictionary HTMLCanvasElementHighDynamicRangeOptions {
    boolean enabled = false;
  }
```

Add a new method to ```HTMLCanvasElement``` to allow configuring HDR.

```idl
  partial interface HTMLCanvasElement {
    void configureHighDynamicRange(HTMLCanvasElementHighDynamicRangeOptions options);
  }
```

### Higher bit storage formats for 2D contexts

Add a new ```CanvasStorageFormat``` enum to allow for higher bit storage formats.

```idl
  enum CanvasStorageFormat {
    'unorm-8',
    'unorm-10-10-10-2',
    'float-16", 
  }
```

Add a ```CanvasStorageFormat``` entry to ```CanvasRenderingContext2DSettings``` to allow 2D rendering contexts to specify their buffer format.

```idl
  partial dictionary CanvasRenderingContext2DSettings {
    CanvasStorageFormat storageFormat = "unorm-8";
  }
```

### Higher bit storage formats for WebGL and WebGPU

WebGL's proposed [``drawingBufferStorage``](https://github.com/KhronosGroup/WebGL/pull/3222) function allows for specifying higher bit depth formats.

WebGPU's ``GPUSwapChainDescriptor`` can allow for specifying higher bit depth formats.

### Color spaces

Update ```PredefinedColorSpace``` to include the following new color spaces.

```idl
  partial enum PredefinedColorSpace {
    'srgb-linear',
    'rec2100-hlg',
    'rec2100-pq',
  }
```

### Conversion between color spaces

All ```PredefinedColorSpace``` are defined by how they are converted to XYZD50 under relative colorimetric intent.

Converting from XYZD50 to a ```PredefinedColorSpace``` is done by performing the inverse of the conversion to XYZD50.

All color space conversions thus have the following properties:

* They are invertible (up to clamping).
* They are context independent.
  * There is one and only one way to convert from one space to another, and it does not depend on the operation being performed.
  * We will allow specific locations in the API where this may be violated.
* They are path independent.
  * Converting from space A to C is the same as converting from space A to B to C.

#### ```srgb-linear```

To convert ```srgb-linear``` to XYZD50 under relative colorimetric intent, perform the following steps:

* Apply the matrix transformation to convert the [sRGB primaries](https://www.w3.org/TR/css-color-4/#valdef-color-srgb) to the XYZ primaries.
* Apply the matrix transformation to convert the [sRGB white point](https://www.w3.org/TR/css-color-4/#valdef-color-srgb) to the D50 white point.

#### ```rec2100-hlg```

To convert ```rec2100-hlg``` to XYZD50 under relative colorimetric intent, perform the following steps:

* Apply the HLG inverse OETF defined in Table 5 of [ITU-R 2100](https://www.itu.int/dms_pubrec/itu-r/rec/bt/R-REC-BT.2100-2-201807-I!!PDF-E.pdf).
* Apply the matrix transformation to convert the primaries specified Table 2 of [ITU-R 2100](https://www.itu.int/dms_pubrec/itu-r/rec/bt/R-REC-BT.2100-2-201807-I!!PDF-E.pdf) to the XYZ primaries.
* Apply the matrix transformation to convert the white point from the reference white specified Table 2 of [ITU-R 2100](https://www.itu.int/dms_pubrec/itu-r/rec/bt/R-REC-BT.2100-2-201807-I!!PDF-E.pdf) to the D50 white point.

If this is followed, then converting from  ```rec2100-hlg``` to any SDR color space will not result in any luminance clipping.
This is a desirable property.

#### ```rec2100-pq```

To convert ```rec2100-pq``` to XYZD50 under relative colorimetric intent, perform the following steps:

* Apply the reference PQ EOTF defined in Table 4 of [ITU-R 2100](https://www.itu.int/dms_pubrec/itu-r/rec/bt/R-REC-BT.2100-2-201807-I!!PDF-E.pdf).
* Apply the matrix transformation to convert the primaries specified Table 2 of [ITU-R 2100](https://www.itu.int/dms_pubrec/itu-r/rec/bt/R-REC-BT.2100-2-201807-I!!PDF-E.pdf) to the XYZ primaries.
* Apply the matrix transformation to convert the white point from the reference white specified Table 2 of [ITU-R 2100](https://www.itu.int/dms_pubrec/itu-r/rec/bt/R-REC-BT.2100-2-201807-I!!PDF-E.pdf) to the D50 white point.

If this is followed, then converting from ```rec2100-pq``` to any SDR color space will result in an undesirably dark image, because 10,000 nits will map to diffuse white.
The alternative is to scale the result so that, say, 100 nits maps to diffuse white.
That will be better in that it will be less undesirably dark, but it will be worse in that it will introduce severe luminance clipping.
There is no way to win here.
The most we can hope for is to make the math easy.

### Compositing the HDR ```HTMLCanvasElement```

When compositing an ```HTMLCanvasElement```, it will be composited in a way that is identical to how an ```<img>``` or ```<video>``` element of the same color space would be composited.

Performing appropriate tone mapping is the responsibility of the browser, the operating system, and the display device.

For the color space ```srgb-linear```, the browser will make its best effort to pass the buffer through directly to the display device, with no additional processing.
Consequently, it is not possible to color match between such buffers and content outside of the canvas.

For the color space ```rec2100-hlg```, the browser will composite the canvas's pixel values identically to how it would composite the pixel values of an image with the same color space.

For the color space ```rec2100-pq```, the browser will composite the canvas's pixel values indentically to how it would composite the pixel values of an image with teh same color space and no metadata.

## Example Applications

### The sRGB linear color space

#### An HDR WebGL application

In this example, a WebGL application enables and HDR default drawing buffer, and clears it to the pixel value ``(1,1,1,1)``.

```javascript
    var canvas = document.getElementById('MyCanvas');
    canvas.configureHighDynamicRange({enabled:true});

    var gl = canvas.getContext('webgl2');
    gl.drawingBufferStorage(gl.RGBA16F, canvas.width, canvas.height);
    gl.colorSpace = 'srgb-linear';
    gl.clearColor(1.0, 1.0, 1.0, 1.0);
    gl.clear(gl.COLOR_BUFFER_BIT);
```

When composited, this canvas is not guaranteed to match any specific color outside of the canvas (it is not guaranteed that this will match the CSS color ```'white'```).

### The ```rec2100-hlg``` color space

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
    canvas.configureHighDynamicRange({enabled:true});

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

This canvas, when composited, will always be identical to displaying the image an ordinary ```<img>``` element.

```xml
  <img src='https://storage.googleapis.com/dalecurtis/cosmos_hlg.avif'/>
```

#### Adding subtitles to an HLG image

Suppose we wish to change the above example to draw subtitles at a brightness that corresponds to an HLG signal value of 0.75.

```javascript
    var canvas = document.getElementById('MyCanvas');
    canvas.configureHighDynamicRange({enabled:true});
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

Note that this is identical to specifying the subtitle color in ```'srgb-linear'```.

```javascript
    context.fillStyle = 'color(srgb-linear 0.265  0.265 0.265)';
```

### The ```rec2100-pq``` color space

In this example, we use a PQ 2D canvas to display a PQ image.

```javascript
    var canvas = document.getElementById('MyCanvas');
    canvas.configureHighDynamicRange({enabled:true});

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

There is no guarantee that this canvas, when composited, will be equivalent to displaying the image an ordinary ```<img>``` element.

```xml
  <img src='https://storage.googleapis.com/dalecurtis/cosmos_hlg.avif'/>
```

The reason this guarantee cannot be made is that the ```<img>``` element may do custom tonemapping based on embedded metadata.
There does not exist any API for extracting this metadata from an ```Image``` object, and thus this metadata cannot be passed on to the ```HTMLCanvasElement```.

## Additions

The above is a simplified version of the API that was proposed earlier.

### HDR compositing independent of color space

There are three big holes in the above API that I have a single proposal to fix.

* There's no way to match SDR colors, and just extend to HDR
* There's no way to have a linear space working space for an HLG or PQ canvas
* We never defined what happens when HDR is enabled for other color spaces like ```'srgb'```.

The fix that I propose to this is to allow ```configureHighDyanmicRange``` to specify the compositing mode.
There would be four compositing modes:

* Passthrough, which is high performance and matches the default for ```srgb-linear``` above
* HLG, which matches the default for ```rec2100-hlg``` above
* PQ, which matches the default for ```rec2100-pq``` above
* Extended, which matches SDR content.

In that case, the code to enable the fast mode would be:

```javascript
    canvas.configureHighDynamicRange({enabled:true, mode:'passthrough'});
    var context = canvas.getContext('2d',
        {colorSpace:'srgb-linear', storageFormat:'float-16'});
```

Or perhaps there would be no ```enabled``` member, and only a ```mode``` member, as in:

```javascript
    canvas.configureHighDynamicRange({mode:'passthrough'});
```

The code to work in a linearized HLG space would be:

```javascript
    canvas.configureHighDynamicRange({mode:'hlg'});
    var context = canvas.getContext('2d',
        {colorSpace:'srgb-linear', storageFormat:'float-16'});
```

### ImageBitmap conversion options

The ```ImageBitmapOptions``` structure already has ```colorSpace``` and ```colorSpaceConversion``` members.

The ```colorSpaceConversion``` has ```default```, which is relative colorimetric intent, and ```none```, which is to simply reinterpret values directly.

We could consider adding a ```perceptual``` option for ```colorSpaceConversion```, which would perform some sort of tonemapping. We could also add a ```bt2408``` option.
