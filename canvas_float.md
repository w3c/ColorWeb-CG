# Canvas Floating Point Color Values

## Introduction

This proposal introduces the ability to use floating-point pixel formats in `CanvasRenderingContext2D` and `OffscreenCanvasRenderingContext2D`.

## Background

## Current capabilities

Both `CanvasRenderingContext2D` and `OffscreenCanvasRenderingContext2D` contain an output bitmap that they render to.

The pixel format of this output bitmap is currently unspecified, but all implementations currently use a 8 bits per channel RGB or RGBA pixel format for this bitmap.

## Use Cases and Motivation

High dynamic range and wide color gamut content content often require more than 8 bits per channel to avoid banding artifacts.

Medical applications (e.g, radiography) demand higher than 8 bits per channel resolution.

Modern high end displays are capable of displaying more than 8 bits per channel.

## Proposed changes

### Changes to `CanvasRenderingContext2DSettings`

Create the new enum type `CanvasColorType`.

```idl
  enum CanvasColorType {
    "unorm8",
    "float16",
  };
```

Add to `CanvasRenderingContext2DSettings` a `colorType` member of type `CanvasColorType`, with a default of `"unorm8"`.

```idl
  partial dictionary CanvasRenderingContext2DSettings {
    CanvasColorType colorType = "unorm8";
  };
```

### Changes to `HTMLCanvasElement` and `OffscreenCanvas` 2D context creation

In 
[`HTMLCanvasElement`'s 2D context creation algorithm](https://html.spec.whatwg.org/multipage/canvas.html#2d-context-creation-algorithm)
and
[`OffscreenCanvas`' 2D context creation algorithm](https://html.spec.whatwg.org/multipage/canvas.html#2d-context-creation-algorithm),
when the `CanvasRenderingContext2DSettings` is created (from the optional `options` parameter),
the value specified in `colorType` will determine the type for the color channels of the rendering context's [output bitmap](https://html.spec.whatwg.org/multipage/canvas.html#output-bitmap) as follows:
* If the color type of the output bitmap is `"unorm8"`, then the color channels of the be 8 bit unsigned.
* If the color type of the output bitmap is `"float16"`, then the output bitmap's precision must be IEEE 754 16-bit floating-point.

### Floating point behavior

Rendering operations to an output bitmap of color type `"float16"` may be subject to the same differences from IEEE 754 behavior that are outlined in [WebGPU's WGSL](https://www.w3.org/TR/WGSL/#differences-from-ieee754).

### Display of values outside of the `[0, 1]` interval

When rendering a canvas to an output device, color values are [converted to the output device's color space using relative colorimetric intent](https://html.spec.whatwg.org/multipage/canvas.html#colour-spaces-and-colour-correction).

Add text clarifying that colors values outside of the standard dynamic range of the output device will be projected to the standard dynamic range of the output device, as is [already specified in WebGPU](https://www.w3.org/TR/webgpu/#gpucanvastonemappingmode).

### Preservation of precision during export

#### To `ImageBitmap`

When converting an output bitmap of color type `"float16"` to an `ImageBitmap` via [`createImageBitmap`](https://html.spec.whatwg.org/multipage/imagebitmap-and-animations.html#dom-createimagebitmap), the resulting `ImageBitmap` must not lose any precision.

#### To encoded image

When [serializing an output bitmap to a file](https://html.spec.whatwg.org/multipage/canvas.html#a-serialisation-of-the-bitmap-as-a-file) (e.g, using `toBlob()` or `toDataURL()`, the output file format will impose restrictions on what can be encoded. To that section, add the following clarifications:

* For image types that do not support storing floating point values, pixel values will be clamped to the range `0.0` through `1.0`.
* For the `image/png` image type, if the output bitmap has color type `"float16"`, then the encoded PNG image must be 16-bit.

#### To `ImageData`

When converting an output bitmap of color type `"float16"` to a `Uint8ClampedArray` via the `getImageData` method, pixel values will be clamped to the [0,1] interval, scaled by `255`, and rounded to the nearest integer value.

## Related specifications and issues

### WebGPU

In WebGPU, floating-point canvas color types are already available.
They may be specified in `GPUCanvasConfiguration` by indicating a `format` of `rgba16float`.

### WebGL

In WebGL, floating-point canvas color types are already available.
They may be specified via `drawingBufferStorage` by indicating a `sizedFormat` of `RGBA16F`.

### Canvas High Dynamic Range

This functionality is a prerequisite for the [Canvas High Dynamic Range proposal](https://github.com/w3c/ColorWeb-CG/blob/master/hdr_html_canvas_element.md).
This functionality was separated off from the Canvas High Dynamic Range proposal.

This specification is consistent with the default treatment of out-of-device-range values in [WebGPU](https://www.w3.org/TR/webgpu/#gpucanvastonemappingmode) and WebGL.
All other tone mapping modes are to be unified between WebGPU, WebGL, and 2D canvas, via updates to the Canvas High Dynamic Range proposal.

### ImageData

Previous versions of this specification included support for floating-point types in `ImageData`.
This has now been moved to [this separate issue](https://github.com/whatwg/html/issues/10856).

`ImageData` is a source type for WebGL's `texImage2D` and related functions, WebGPU's `GPUCopyExternalImageSourceInfo` and related functions, and the `createImageBitmap` function.
All new types supported by `ImageData` must be supported and tested in the context of all of these APIs.

## Remarks on choices

### Color type versus pixel format

This proposal uses the term "color type" instead of "pixel format" intentionally to indicate just the data type of the color channels of the format, and not their layout.

The following are problems that this choice avoids:
* A user agent may choose "bgra8unorm" or "rgba8unorm" as the representation of an 8 bit per channel canvas.
  This this implementation detail is now hidden.
* The incorporation of the `alpha` boolean can be done in many different ways.
  E.g, WebGL has `RGB8` and `RGBA8`.
  E.g, WebGPU does has an implicit RGBX behavior via `GPUCanvasAlphaMode` of `opaque`.
  These details are avoided by hiding this detail.

In the future, it may be that we will want to add a format like WebGPU's `rgb10a2unorm`.
This format is different in that the `CanvasColorType` would need to indicate RGB and A separately.
This could be done by via `{colorType:"unorm10-unorm2alpha"}` or `{colorType:"unorm10", alpha:false}`.

### Choice of `"float16"` for `CanvasColorType`

The ability to texture from and render to 16 bit floating-point is universal among modern GPUs.

### Choice of default behavior of `getImageData`

Historically, all calls to `getImageData` have returned an `ImageData` with a `Uint8ClampedArray`.
Changing the default type that is returned by this function will break any software that relies on
that default type, which is currently all software that uses this function.

There is no plan to add floating-point support for `getImageData`.

A new API, `getImageDataAsync`, which takes an already-created `ImageData` as a parameter is being developed.
By its construction, this API does not suffer from any ambiguity about default parameters.

See [this issue](https://github.com/whatwg/html/issues/10855) for details.

### Choice of image encoding behaviors

It is possible to encode pixel values outside of the [0,1] interval using HDR color spaces (e.g, Rec2100 PQ and Rec2100 HLG).

Do not exploit this capability yet.
Keep the existing behavior of ensuring of having the encoded image have the same color space as the canvas output bitmap.

Future work that adds HDR color spaces (e.g: `rec2100-linear`) will default to using Rec2100 PQ encoding, preserving this behavior.

### Feature detection

The presence of this feature can be detected by checking for the presence of `colorType` in the `CanvasRenderingContext2DSettings` returned by `getContextAttributes()`.

This method is not present for `OffscreenCanvasRenderingContext2D`, but that is likely [a spec oversight](https://github.com/whatwg/html/issues/10872).
