# WebGPU texture import and copy HDR headroom parameter

## Introduction

This proposal suggest a mechanism for specifying the amout of tone mapping to perform (or not) when importing HDR images and video to WebGPU textures.

## Related work and motivation

HDR images can be drawn an infinite number of ways, often unrepresentable as a single bitmap (e.g, gainmap images).

This is part of the general problem wherein an HDR image must be put into pixels in a buffer.
At the moment the HDR image is put into pixels in a buffer, this infinite number of ways of representing it must be collapsed into a single representation.

A related problem is when an HDR image must be drawn to a `CanvasRenderingContext2D` via `drawImage`.
The analogous proposal to this proposal is to add a target HDR headroom parameter.
See [this explainer](https://github.com/ccameron-chromium/ColorWeb-CG/blob/canvas2d_hdr_headroom/canvas2d_hdr_headroom.md) and [this spec issue](https://github.com/whatwg/html/issues/11165).

## Proposal

### Copying external images

To the [`GPUCopyExternalImageDestInfo`](https://www.w3.org/TR/webgpu/#gpucopyexternalimagedestinfo) dictionary, add the following entries:

```idl
  partial dictionary GPUCopyExternalImageDestInfo {
    // The HDR headroom (in log2 space) to tone map the source image to when
    // copying to the destination texture.
    float hdrHeadroom = 0;
  }
```

This parameter is the same as the targeted HDR headroom from the [2D canvas explainer](https://github.com/ccameron-chromium/ColorWeb-CG/blob/canvas2d_hdr_headroom/canvas2d_hdr_headroom.md).
The value must be greater or equal to zero.

A value of `0` (the default) indicates that tone mapping to SDR should be performed.
Applications that do not indend to handle HDR content can safely use this value.

A value of `Infinity` indicates that no tone mapping is to be performed.

The tone mapping of HDR content depends on the content.
Initial tests will use ISO 21496-1 gain map images, which has a well-defined tone mapping parameterized by HDR headroom.
The default tone mapping to use for HDR content is out of the scope of this proposal.

### Importing external images

To the [`GPUExternalTextureDescriptor`](https://www.w3.org/TR/webgpu/#external-texture-creation) dictionary, add the following entries:

```idl
  partial dictionary GPUExternalTextureDescriptor {
    // The HDR headroom (in log2 space) to tone map the source image to when
    // sampling from the image.
    float hdrHeadroom = 0;
  }
```

Note that this will require tone mapping be performed on the fly.

