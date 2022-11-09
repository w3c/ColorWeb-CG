# Canvas Floating Point Color Values

## Introduction

This proposal introduces the ability to use floating-point pixels formats `CanvasRenderingContext2D`, `OffscreenCanvasRenderingContext2D`, and `ImageData`.

## Background

## Current capabilities

Both `CanvasRenderingContext2D` and `OffscreenCanvasRenderingContext2D` have an output bitmap that they render to.
The pixel format of this output bitmap is currently unspecified.
Many implementations use an 8 bits per channel RGB or RGBA pixel format for this bitmap (this is likely the case for all implementations, but the author has not examined all implementations).

An `ImageData` has a `data` member, which is a `Uint8ClampedArray`, making its format 8 bits per channel RGBA.

## Use Cases and Motivation

High dynamic range and wide color gamut content often requires more than 8 bits per channel to avoid banding artifacts.

Medical applications (e.g, radiography) demand higher than 8 bits per channel resolution.

Modern high end displays are capable of displaying more than 8 bits per channel.

## Proposed changes

### Changes to `CanvasRenderingContext2DSettings`

Create a new `CanvasColorType` enum to specify the type for the color channels of the output bitmap of a `CanvasRenderingContext2D` and `OffscreenCanvasRenderingContext2D`.

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
Color values that specify a brightness outside of the standard dynamic range will have their brightness limited to the standard dynamic range of the output device, unless the canvas is explicitly indicated to be high dynamic range (and this proposal intentionally does not include this mechanism).

### Changes to `ImageData`

Create a new `ImageDataColorType` enum to specify the type for the color channels of an `ImageData`.

```idl
  enum ImageDataColorType {
    "uint8Clamped",
    "float32",
  };
```

Add to `ImageDataSettings` an `ImageDataColorType` member, to specify this color type.

```idl
  partial dictionary ImageDataSettings {
    ImageDataColorType colorType = "uint8Clamped";
  };
```

Change `ImageData` to allow the `data` member to be either `Uint8ClampedArray` or `Float32Array` via a union type `ImageDataArray`.

```idl
  typedef (Uint8ClampedArray or Float32Array) ImageDataArray;

  partial interface ImageData {
    readonly attribute ImageDataArray data;
  }
```

The type of an `ImageData`'s `data` member is determined at creation time by the `colorType` value specified in `ImageDataSettings` as follows:

* If `colorType` is `"uint8Clamped"` then `data` is of type `Uint8ClampedArray`.
* If `colorType` is `"float32"` then `data` is of type `Float32Array`.

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

### The names `"uint8Clamped"` and `"unorm8"`

An alternative would be to use the value `"uint8"` for both `ImageDataColorType` and `CanvasColorType`.
The alternative may be simpler to reason about, or it may be a source of confusion.
