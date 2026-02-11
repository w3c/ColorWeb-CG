# WebGL texture import HDR headroom parameter

## Introduction

This proposal suggests a mechanism for specifying the amout of tone mapping to perform (or not) when importing HDR images and video to WebGL textures.

This fits into the [HDR on the web, the big picture](hdr-big-picture.md) proposal. It is the "red star number 2" problem in the proposal diagram.

## Motivation

See [related section for 2D canvas](canvas-compositing-headroom.md#motivation)

## Related non-web APIs

See [related section for 2D canvas](canvas-compositing-headroom.md#related-non-web-apis)

## Proposal

To the [`WebGLRenderingContextBase`](https://registry.khronos.org/webgl/specs/latest/1.0/#5.14) interface, add the following attribute:

```idl
  partial interface WebGLRenderingContextBase {
    // The linear HDR headroom to tone map the source image to when
    // copying to the destination texture using tex[Sub]Image.
    attribute float unpackLinearHDRHeadroom = 1;
  }
```

Note the similarity between this attribute and the [`unpackColorSpace`](https://registry.khronos.org/webgl/specs/latest/1.0/#5.14.1) attribute.
This attribute shall be ignored in all situations where the `unpackColorSpace` is ignored (when `UNPACK_COLORSPACE_CONVERSION_WEBGL` is `NONE`).

This attribute may be set to `Infinity` to indicate that the "maximum headroom" version is desired.

Setting this attribute to value outside of the interval [`1`, `Infinity`] shall have no effect.

## Testing

Testing should use the same set of input images used in the [related section for 2D canvas](canvas-compositing-headroom#testing).

