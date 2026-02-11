# WebGPU texture import and copy HDR headroom parameter

## Introduction

This proposal suggest a mechanism for specifying the amout of tone mapping to perform (or not) when importing HDR images and video to WebGPU textures and shaders.

This fits into the [HDR on the web, the big picture](hdr-big-picture.md) proposal. It is the "red star number 3" and "red star number 4" problems in the proposal diagram.

## Motivation

See [related section for 2D canvas](canvas-compositing-headroom.md#motivation)

## Related non-web APIs

See [related section for 2D canvas](canvas-compositing-headroom.md#related-non-web-apis)

## Proposal

### Copying external images

To the [`GPUCopyExternalImageDestInfo`](https://www.w3.org/TR/webgpu/#gpucopyexternalimagedestinfo) dictionary, add the following entry:

```idl
  partial dictionary GPUCopyExternalImageDestInfo {
    // The linear HDR headroom to tone map the source image to when
    // copying to the destination texture.
    float linearHDRHeadroom = 1;
  }
```

To the [`GPUCopyExternalImageDestInfo`](https://www.w3.org/TR/webgpu/#gpucopyexternalimagedestinfo) dictionary, add the following entry:

```idl
  partial dictionary GPUExternalTextureDescriptor {
    // The linear HDR headroom to tone map the source image to when
    // sampling from the image.
    float linearHDRHeadroom = 1;
  }
```

Setting this attribute to value outside of the interval [`1`, `Infinity`] shall throw an exception.


