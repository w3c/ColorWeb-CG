# WebGL texture import HDR headroom parameter

## Introduction

This proposal suggest a mechanism for specifying the amout of tone mapping to perform (or not) when importing HDR images and video to WebGL textures.

## Related work and motivation

HDR images can be drawn an infinite number of ways, often unrepresentable as a single bitmap (e.g, gainmap images).

This is part of the general problem wherein an HDR image must be put into pixels in a buffer.
At the moment the HDR image is put into pixels in a buffer, this infinite number of ways of representing it must be collapsed into a single representation.

A related problem is when an HDR image must be drawn to a `CanvasRenderingContext2D` via `drawImage`.
The analogous proposal to this proposal is to add a target HDR headroom parameter.
See [this explainer](https://github.com/ccameron-chromium/ColorWeb-CG/blob/canvas2d_hdr_headroom/canvas2d_hdr_headroom.md) and [this spec issue](https://github.com/whatwg/html/issues/11165).

Another related problem is when an HDR image must be imported into WebGPU.
The analogous proposal (which has a lot of the exact same language as this proposal) is in [this explainer](https://github.com/ccameron-chromium/ColorWeb-CG/blob/canvas2d_hdr_headroom/webgpu_hdr_headroom.md).

## Proposal

To the [`WebGLRenderingContextBase`](https://registry.khronos.org/webgl/specs/latest/1.0/#5.14) interface, add the following attribute:

```idl
  partial interface WebGLRenderingContextBase {
    // The HDR headroom (in log2 space) to tone map the source image to when
    // copying to the destination texture using tex[Sub]Image.
    attribute float unpackHDRHeadroom = 0;
  }
```

This parameter is the same as the targeted HDR headroom from the [2D canvas explainer](https://github.com/ccameron-chromium/ColorWeb-CG/blob/canvas2d_hdr_headroom/canvas2d_hdr_headroom.md) and the [WebGPU explainer](https://github.com/ccameron-chromium/ColorWeb-CG/blob/canvas2d_hdr_headroom/webgpu_hdr_headroom.md).

Note the similarity between this attribute and the `unpackColorSpace` attribute.
This attribute shall be ignored in all situations where the `unpackColorSpace` is ignored (when `UNPACK_COLORSPACE_CONVERSION_WEBGL` is `NONE`).

