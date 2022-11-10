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

Create a new enum `ImageDataDataType` to specify the type for the color channels of an `ImageData`.

```idl
  enum ImageDataDataType {
    "uint8Clamped",
    "float32",
  };
```

Add to `ImageDataSettings` a `ImageDataDataType` member, to specify this color type.

```idl
  partial dictionary ImageDataSettings {
    ImageDataDataType dataType;
  };
```

Change `ImageData` to allow the `data` member to be either `Uint8ClampedArray` or `Float32Array` via a union type `ImageDataArray`.

```idl
  typedef (Uint8ClampedArray or Float32Array) ImageDataArray;

  partial interface ImageData {
    readonly attribute ImageDataArray data;
  }
```

The type of an `ImageData`'s `data` member is determined at creation time by the `dataType` value specified in `ImageDataSettings` as follows:

* If `dataType` is unspecified or is `"uint8Clamped"` then `data` is of type `Uint8ClampedArray`.
* If `dataType` is `"float32"` then `data` is of type `Float32Array`.

### Changes to `CanvasImageData` interface

In the interface `CanvasImageData`, the functions `createImageData` and `getImageData` will create an `ImageData`, and also optionally take an `ImageDataSettings`.

If the `ImageDataSettings` specifies `dataType`, then the resulting `ImageData`'s `data`'s type will be as described above.

If the `ImageDataSettings` does not specify `dataType`, then the resulting `ImageData`'s `data`'s type will be as follows:

* If the `CanvasImageData` has a `colorType` of `"unorm8"`, then the `ImageData`'s `data` will be of type `Uint8ClampedArray`.
* If the `CanvasImageData` has a `colorType` of `"float16"`, then the `ImageData`'s `data` will be of type `Float32Array`.

For the function `createImageData` that takes an `ImageData` as an argument, the resulting `ImageData`'s `data` should have the same type as the input `ImageData`.

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
This can be avoided by examining the type of the `ImageData`' `data` member.

### Choice of defaulting `getImageData` to `Float32Array` for `"float16"` canvas

There exists a trade-off in the default behavior of `getImageData` for a `"float16"` canvas.

Suppose one is to default to returning a `Uint8ClampedArray`.
The benefit of this behavior is that there is a lesser likelihood of falling into the above mentioned pitfall.
The downside of this behavior is that a `getImageData` and then `putImageData` round-trip will lose precision.

Suppose one is to default to returning a `Float32Array`.
The benefit of this behavior is that a `getImageData` and then `putImageData` round-trip will not lose precision.
The downside is that there is a higher likelihood of falling into the above mentioned pitfall.

This is resolved in favor of preserving round-trip precision, following the decision with respect to the default behavior for `colorSpace` for `getImageData`.

### The names `"uint8Clamped"` and `"unorm8"`

An alternative would be to use the value `"uint8"` for both `ImageDataDataType` and `CanvasColorType`.
The alternative may be simpler to reason about, or it may be a source of confusion.

