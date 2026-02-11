# 2D Canvas drawing HDR headroom parameter

## Introduction

This proposal suggest a mechanism for drawing HDR content as HDR to a `CanvasRenderingContext2D` or `OffscreenCanvasRenderingContext2D`.

This fits into the [HDR on the web, the big picture](hdr-big-picture.md) proposal. It is the "red star number 1" problem in the proposal diagram.

## Motivation

### Remove ambiguity

When drawing an image to an HDR canvas, using `drawImage`, the source image is converted from its native color space to the color space of the canvas.
There is currently no indication about what tone mapping, if any, should be done during this conversion.

### Enable drawing HDR images and video as HDR

By default, all implementations tone map to SDR in the process of converting to the color space of the canvas.
It should be possible to preserve HDR-ness of HDR images in the canvas, especially now that canvas' backing bitmaps can be floating-point.

### Enable drawing HDR CSS colors as HDR

HDR CSS colors (using [`color-hdr`](https://drafts.csswg.org/css-color-hdr/#hdr-color-function)) are parameterized by a target HDR headroom.
When such colors are used with 2D canvas, this displaying their non-HDR representation.

### Standardize tone mapping for SDR renditions of HDR images

This behavior can be used to ensure that all browsers tone map HDR images to SDR in the same way.

### Ubiquitous HDR images

There is an enormous and growing corpus of ISO 21496-1 HDR gain map images being produced, as this is now the default format for many popular phones.

## Related non-web APIs

### Chromium internals

The Chromium browsers have a piece of rendering state called the [target HDR headroom](https://source.chromium.org/chromium/chromium/src/+/main:cc/paint/paint_op_buffer.h;l=169;drc=87c3217dc3fec0f441b68f33d339b7f3a707b11d) that is set during rendering.
When drawing an HDR image using `drawImage`, that parameter is used for tone mapping by Skia.

### CoreGraphics API

In CoreGraphics, there exists a similar API called [`CGContextSetEDRTargetHeadroom`](https://developer.apple.com/documentation/coregraphics/cgcontext/setedrtargetheadroom(_:)?language=objc), which is documented (in the header file) as "Set target EDR headroom on a context to be used when rendering HDR content to the context.".

These two implementations are extremely similar, down to the variable names.

## Separate work

### Tone mapping at display time

The HDR headroom used when rendering is completely independent from the tone mapping behavior.
In practice, most applications will want to tone map taking into account the HDR headroom that was used for displaying the canvas.

A [separate set of proposals](hdr-big-picture.md#displaying-an-hdr-canvas) address that issue.

## Proposal

Add to the mixin interface, `CanvasCompositing` the following attribute

```idl
  partial interface mixin CanvasCompositing {
    // The linear HDR headroom to tone map content to when
    // drawing to the canvas.
    attribute unrestricted double globalLinearHDRHeadroom; // (default 1)
  }
```

This attribute may be set to `Infinity` to indicate that the "maximum headroom" version is desired.

Setting this attribute to value outside of the interval [`1`, `Infinity`] will have no effect.

### Interactions with drawing

The attribute will affect the rendering of images when drawn with `drawImage`.

It will also affect the tone mapping of CSS colors specified by `color-hdr`.

This will not affect how CSS colors specified without `color-hdr` are drawn to the canvas.
E.g, the color `color(srgb 2 2 2)` will always write the value [2,2,2] to a canvas with `colorSpace` `srgb` and `colorType` `float16`.

## Testing

The initial tests for this image type should be HDR images that have tone mapping that is parameterized by HDR headroom.

The most clearly specified version of this is ISO 21496-1 gain map images.
A set of WPT tests that draw gain map images at multiple HDR headrooms should be added.

A less clearly specified version of this is ISO 22028-5 images.
The tone mapping at HDR headroom of 0 (SDR) is defined in ISO 22208-5, but tone mapping to other headrooms is not specified.
A set of WPT tests that draw these images at HDR headroom 0 (SDR) and their maximum HDR headroom should be added.

