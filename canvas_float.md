# Canvas Floating Point Color Values

## Introduction

This proposal introduces the ability to use floating-point pixel formats in `CanvasRenderingContext2D`, `OffscreenCanvasRenderingContext2D`, and `ImageData`.

## Background

## Current capabilities

Both `CanvasRenderingContext2D` and `OffscreenCanvasRenderingContext2D` contain an output bitmap that they render to.
The pixel format of this output bitmap is currently unspecified.
Many implementations use a 8 bits per channel RGB or RGBA pixel format for this bitmap (this is likely the case for all implementations, but the author has not examined all implementations).

An `ImageData` has a `data` member, which is a `Uint8ClampedArray`, making its format 8 bits per channel RGBA.

## Use Cases and Motivation

High dynamic range and wide color gamut content content often require more than 8 bits per channel to avoid banding artifacts.

Medical applications (e.g, radiography) demand higher than 8 bits per channel resolution.

Modern high end displays are capable of displaying more than 8 bits per channel.

## Proposed changes

### Changes to `CanvasRenderingContext2DSettings`

Create the new enum `CanvasColorType` to specify the type for the color channels of the output bitmap of a `CanvasRenderingContext2D` and `OffscreenCanvasRenderingContext2D`.

```idl
  enum CanvasColorType {
    "unorm8",
    "float16",
  };
```

Add to `CanvasRenderingContext2DSettings` a `CanvasColorType` member, to specify this color type.

```idl
  partial dictionary CanvasRenderingContext2DSettings {
    CanvasColorType colorType = "unorm8";
  };
```

The value specified in `colorType` will determine the type for the color channels of the output bitmap.

When a canvas has a color type of `"float16"` color values may be outside of the range `[0, 1]`.
Such values may be used to specify colors outside of the gamut determined by the canvas' color space's primaries.

When rendering `"float16"` color values to an output device, color values will be converted to the output device's color space using relative colorimetric intent.
Colors values that specify a brightness outside of the standard dynamic range will have their brightness limited to the standard dynamic range of the output device, unless the canvas is explicitly indicated to be high dynamic range (and this proposal intentionally does not include this mechanism).

### Changes to `ImageData`

Create a new enum `ImageDataColorType` to specify the type for the color channels of an `ImageData`.

```idl
  enum ImageDataColorType {
    "unorm8",
    "float32",
  };
```

Add to `ImageDataSettings` a `ImageDataColorType` member, to specify this color type.

```idl
  partial dictionary ImageDataSettings {
    ImageDataColorType colorType;
  };
```

Change `ImageData` to allow the `data` member to be either `Uint8ClampedArray` or `Float32Array` via a union type `ImageDataArray`.

```idl
  typedef (Uint8ClampedArray or Float32Array) ImageDataArray;

  partial interface ImageData {
    readonly attribute ImageDataColorType colorType;
    readonly attribute ImageDataArray data;
  }
```

In the constructor for `ImageData`, if an `ImageDataSettings` is specified, and that `ImageDataSettings` has a `colorType`, then the created `ImageData` will have the specified value for its `colorType`.
If no `ImageDataSettings` is specified, or no `colorType` is specified, then the created `ImageData` will have `colorType` `"unorm8"`.

The type of an `ImageData`'s `data` member is determined by its `colorType` member as follows:

* If `colorType` is `"unorm8"` then `data` is of type `Uint8ClampedArray`.
* If `colorType` is `"float32"` then `data` is of type `Float32Array`.

### Changes to `CanvasImageData` interface

In the interface `CanvasImageData`, the functions `createImageData` and `getImageData` will create an `ImageData`, and also optionally take an `ImageDataSettings`.

If the `ImageDataSettings` specifies `colorType`, then the resulting `ImageData` will have that specified `colorType`.

If the `ImageDataSettings` does not specify `colorType`, then the resulting `ImageData`'s `colorType` will be `"unorm8"`.

### Values outside of the `[0, 1]` interval

All color spaces supported by `PredefinedColorSpace` are defined for all real numbers.

Both `"srgb"` and `"display-p3"` are extended using point symmetry around `0`.
For the precise definition, see [the CSS definition of `"srgb"`](https://www.w3.org/TR/css-color-4/#predefined-sRGB).

### Type precision

The exact storage format of the output bitmap of a `CanvasRenderingContext2D` and `OffscreenCanvasRenderingContext2D` is not directly observable.
There exists no API through which the raw storage may be examined.

If the color type of the output bitmap is `"unorm8"`, then the precision must be at least 8 bits per color channel.
Stored values less than 0 or greater than 255 must be clamped to 0 and 255 respectively.

If the color type of the output bitmap is `"float16"`, then the precision must be at least IEEE 754-2008 16-bit floating-point.

### Type conversions

When reading an output bitmap of color type `"unorm8"` to a `Float32Array`, the resulting values will be normalized from the range `0` through `255` to the range `0.0` through `1.0`.

When converting an output bitmap of color type `"float16"` to a `Uint8ClampedArray`, values less than `0.0` will be clamped to `0`, values greater than `1.0` will be clamped to `255`, and values in the range `0.0` through `1.0` will be scaled by `255` and rounded to the nearest integer value.

When exporting an output bitmap of color type `"float16"` to an data URL or a `Blob` in a format that does not support floating point storage, values outside of the range `0.0` through `1.0` will be clamped.

## Related specifications

### WebGPU

In WebGPU, floating-point canvas color types are already available.
They may be specified in `GPUCanvasConfiguration` by indicating a `format` of `rgba16float`.

### WebGL

In WebGL, floating-point canvas color types are proposed via [drawingBufferStorage](https://github.com/KhronosGroup/WebGL/pull/3222).

### Canvas High Dynamic Range

This functionality is a prerequisite for the [Canvas High Dynamic Range proposal](https://github.com/w3c/ColorWeb-CG/blob/master/hdr_html_canvas_element.md).

This functionality was separated off from the Canvas High Dynamic Range proposal.

## Examples

### Writing the same color in different ways

The following example draws the color `color(display-p3 1 0 0)` in multiple ways.

```javascript
    let canvas = document.getElementById('MyCanvas');

    let context = canvas.getContext('2d', {colorSpace:"display-p3"});
    console.log(context.getContextAttributes().colorSpace) // Prints "display-p3"
    console.log(context.getContextAttributes().colorType)  // Prints "unorm8"

    // Draw using CSS colors.
    context.fillStyle = "color(display-p3 1 0 0)";
    context.fillRect(0, 0, 1, 1);

    // Draw using a floating-point sRGB ImageData, using values outside of the
    // [0, 1] interval.
    let imageDataSRGB = new ImageData(1, 1, {colorType:"float32"});
    console.log(imageDataSRGB.colorSpace); // prints "srgb"
    console.log(imageDataSRGB.colorType);  // prints "float32"
    imageData.data[0] =  1.0930663624351615;
    imageData.data[1] = -0.22674197356975417;
    imageData.data[2] = -0.15013458093711934;
    imageData.data[3] =  1.0;
    context.putImageData(imageDataSRGB, 1, 0);

    // Draw using a floating-point Display P3 ImageData.
    let imageDataDisplayP3 = new ImageData(1, 1, {colorSpace:"display-p3", colorType:"float32"});
    console.log(imageDataDisplayP3.colorSpace); // prints "display-p3"
    console.log(imageDataDisplayP3.colorType);  // prints "float32"
    imageData.data[0] = 1.0;
    imageData.data[1] = 0.0;
    imageData.data[2] = 0.0;
    imageData.data[3] = 1.0;
    context.putImageData(imageDataDisplayP3, 2, 0);

    // Read back the result.
    let imageDataReadback = context.getImageData(0, 0, 3, 1);
    console.log(imageDataReadback.colorSpace); // prints "display-p3"
    console.log(imageDataReadback.colorType);  // prints "unorm8"
    console.log(imageDataReadback.data);       // prints [255, 0, 0, 255 ...] three times.
```

### Reading back contents

The following example draws the color `color(display-p3 1 0 0)` and reads it back.

```javascript
    let canvas = document.getElementById('MyCanvas');

    let context = canvas.getContext('2d', {colorSpace:"srgb", colorType:"float16"});
    console.log(context.getContextAttributes().colorSpace) // Prints "srgb"
    console.log(context.getContextAttributes().colorType)  // Prints "float16"

    // Draw using CSS colors.
    context.fillStyle = "color(display-p3 1 0 0)";
    context.fillRect(0, 0, 1, 1);

    // Read back with default parameters. Note that the resulting values are
    // outside of the [0, 1] interval.
    let imageDataDefault = context.getImageData(0, 0, 1, 1);
    console.log(imageDataDefault.colorSpace); // prints "srgb"
    console.log(imageDataDefault.colorType);  // prints "float32"
    console.log(imageDataDefault.data);       // prints [1.0931, -0.2267, -0.1501, 1.0]

    // Read back forcing colorType to "unorm8". Results are clamped.
    let imageDataUnorm8 = context.getImageData(0, 0, 1, 1, {colorType:"unorm8"});
    console.log(imageDataUnorm8.colorSpace); // prints "srgb"
    console.log(imageDataUnorm8.colorType);  // prints "unorm8"
    console.log(imageDataUnorm8.data);       // prints [255, 0, 0, 255]

    // Read back forcing colorSpace to "display-p3". The result has colorType
    // "float32" by default.
    let imageDataDisplayP3 = context.getImageData(0, 0, 1, 1, {colorSpace:"display-p3"});
    console.log(imageDataDisplayP3.colorSpace); // prints "display-p3"
    console.log(imageDataDisplayP3.colorType);  // prints "float32"
    console.log(imageDataDisplayP3.data);       // prints [1.0, 0.0, 0.0, 1.0]
```

### Determining ImageData type

Consider the function `setToGradient` that initializes an `ImageData` to a horizontal gradient.
This shows how to handle different `ImageData` types.

```javascript
    function setToGradient(imageData) {
      for (var x = 0; x < imageData.width; ++x) {
        for (var y = 0; y < imageData.height; ++y) {
          let offset = 4 * x * imageData.height;
          switch (imageData.colorType) {
            case "unorm8":
              imageData.data[offset+0] = 
              imageData.data[offset+1] = 
              imageData.data[offset+2] = 255 * x / (imageData.width - 1);
              imageData.data[offset+3] = 255;
              break;
            case "float32":
              imageData.data[offset+0] = 
              imageData.data[offset+1] = 
              imageData.data[offset+2] = x / (imageData.width - 1);
              imageData.data[offset+3] = 1.0;
              break;
            default:
              throw("Unexpected ImageData colorType " + imageData.colorType);
          }
        }
      }
    }
```

## Remarks on choices

### Color type versus pixel format

This proposal uses the term "color type" instead of "pixel format" intentionally to indicate just the precision of the color channels of the format, and not their layout.
For example, a user agent may choose BGRA or RGBA as the representation of an 8 bit per channel canvas, and this implementation detail is hidden.

### Choice of `"float16"` for `CanvasColorType`

The ability to texture from and render to 16 bit floating-point is universal among modern GPUs.

### Choice of `Float32Array` for `ImageData`

There does not exist a `Float16Array` type, and so it is not an option.
The only other floating-point type, `Float64Array`, is of unnecessarily high precision.

### Pitfall of `Float32Array` for `ImageData`

There exists a potential pitfall wherein a naive user of `ImageData` may write values intended for a `Uint8ClampedArray` to a `Float32Array`
E.g, one would write `255` instead of `1.0`.

This can be avoided by examining the `colorType` member the `ImageData`.

### Choice of defaulting `getImageData` to `"unorm8"`

Historically, all calls to `getImageData` have returned an `ImageData` with a `Uint8ClampedArray`.

Changing the default type that is returned by this function will break any software that relies on
that default type, which is currently all software that uses this function.

Changing the default type that is returned by this function will significantly hinder the adoption
of `"float16"` canvas, because no application can change the backing of any canvas without first
ensuring that all libraries that it uses have been updated to support all possible return values.

