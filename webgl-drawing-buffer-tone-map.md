# Tone mapping for WebGL

## Introduction

This proposal suggests a mechanism to indicate that the bitmap of a `WebGLRenderingContextBase` is HDR, and that it is to be rendered as HDR.

This fits into the [HDR on the web, the big picture](hdr-big-picture.md) proposal. It is the "magenta sun number 5" and "magenta sun number 6" problem in the proposal diagram.

This builds on the common framework proposed in [2D canvas](canvas-tone-map.md).

## Proposal

To the [`WebGLRenderingContextBase`](https://registry.khronos.org/webgl/specs/latest/1.0/#5.14) interface, add the following function:

```idl
  partial interface WebGLRenderingContextBase {
    // If `toneMapping` is specified, then set the tone mapping of the
    // canvas' output bitmap to `toneMapping`. Return the tone mapping
    // of the canvas' output bitmap.
    attribute CanvasToneMapping drawingBufferToneMapping(optional CanvasToneMapping toneMapping)
  }
```

Note that this function provides setter and getter functionality.
If specified with a `toneMapping` parameter, then it will operate as a setter.
If specified with no parameter, then it will operate as a getter.
This is because, unlike `drawingBufferColorSpace`, the value is a dictionary,
This is similar to the `drawingBufferStorage` function.

