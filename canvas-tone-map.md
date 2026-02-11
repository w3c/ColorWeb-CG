# Tone mapping for 2D Canvas

## Introduction

This proposal suggests a mechanism to indicate that the bitmap of a `CanvasRenderingContext2D` or `OffscreenCanvasRenderingContext2D` is HDR, and that it is to be rendered as HDR.

This fits into the [HDR on the web, the big picture](hdr-big-picture.md) proposal. It is the "magenta sun number 5" and "magenta sun number 6" problem in the proposal diagram.

This is a minimal proposal that is required to unify 2D canvas, WebGL, and WebGPU.
This proposal can be expanded [as indicated in future directions](#future-directions) and in the [SMPTE ST 2094-50 interactions](canvas-smpte-st-2094-50.md) explainer.

### Motivation

The `CanvasRenderingContext2D` or `OffscreenCanvasRenderingContext2D` [output bitmap](https://html.spec.whatwg.org/multipage/canvas.html#output-bitmap) can represent arbitrary color values by using [floating point](https://html.spec.whatwg.org/multipage/canvas.html#dom-canvascolortype-float16) pixel values.

These pixel values may be used to represent higher precision SDR content or HDR content.
The specification currently does not indicate any way to distinguish between these two uses.
Nor does the specification have any place to indicate a specific tone mapping mechanism.

### Related work

This problem has existed for WebGPU since its creation, becuase WebGPU has always allowed floating point output buffers via its [`GPUCanvasConfiguration`](https://www.w3.org/TR/webgpu/#dictdef-gpucanvasconfiguration) [`format`](https://www.w3.org/TR/webgpu/#dom-gpucanvasconfiguration-format) parameter.

WebGPU has solved this problem (with an eye towards solving it for 2D canvas and WebGL) via its
[`GPUCanvasConfiguration`](https://www.w3.org/TR/webgpu/#dictdef-gpucanvasconfiguration)
[`toneMapping`](https://www.w3.org/TR/webgpu/#dom-gpucanvasconfiguration-tonemapping) parameter.

This parameter is a dictionary of type [`GPUCanvasToneMapping`](https://www.w3.org/TR/webgpu/#dictdef-gpucanvastonemapping).
At present, the only member is a [`GPUCanvasToneMappingMode`](https://www.w3.org/TR/webgpu/#gpucanvastonemappingmode]) enum.

Please read the documentation of the two values ([`standard`](https://www.w3.org/TR/webgpu/#dom-gpucanvastonemappingmode-standard) and [`extended`](https://www.w3.org/TR/webgpu/#dom-gpucanvastonemappingmode-extended)), since it is precise.

## Proposal

### IDL changes

Rename `GPUCanvasToneMapping` to `CanvasToneMapping`, and move it from the WebGPU specification to the HTML specification.
The resulting interface would be:

```idl
  dictionary CanvasToneMapping {
    CanvasToneMappingMode mode = "standard";
  }; 
```

Rename `GPUCanvasToneMappingMode` to `CanvasToneMappingMode`, and move it from the WebGPU specification to the HTML specification.
The resulting interface would be:

```idl
  enum CanvasToneMappingMode {
      "standard",
      "extended",
  };
```

To the `CanvasRenderingContext2DSettings` dictionary, add the following entry:

```idl
  partial dictionary CanvasRenderingContext2DSettings {
    CanvasToneMapping toneMapping = {mode:"standard"};
  }
```

This indicates that the default behavior is to restrict color values to the standard dynamic range of the target display, which is the current behavior of all browsers.

### Interactions with `dynamic-range-limit`

The `dynamic-range-limit` applies to HTML cavas elements the same as it applies to video and image elements.
A canvas that specifies to use `"extended"`, but has `dynamic-range-limit:standard` will display the same the canvas would have had it specified `"standard"`, because the "tone mapping" is just to clamp (or project, more accurately) to the available dynamic range.
For a non-trivial tone mapping, this would not be true.

### Interactions with `ImageBitmap`

When an `ImageBitmap` is created from an `HTMLCanvasElement` or `OffscreenCanvas`
by any mechanism
(including
[`createImageBitmap`](https://html.spec.whatwg.org/multipage/imagebitmap-and-animations.html#dom-createimagebitmap)
and
[`transferToImageBitmap`](https://html.spec.whatwg.org/multipage/canvas.html#dom-offscreencanvas-transfertoimagebitmap)),
the resulting `ImageBitmap` shall preserve the behavior specified by `CanvasToneMapping`.

This includes when it is displayed as an [`ImageBitmapRenderingContext`](https://html.spec.whatwg.org/multipage/canvas.html#the-imagebitmaprenderingcontext-interface),
and when it is used as a [`CanvasImageSource`](https://html.spec.whatwg.org/multipage/canvas.html#canvasimagesource) or similar.

### Interactions with `VideoFrame`

When a `VideoFrame` is created from an `HTMLCanvasElement` or `OffscreenCanvas`,
the resulting object should include [metadata](https://www.w3.org/TR/webcodecs-video-frame-metadata-registry/)
indicating the tone mapping specified.

### Serialization to a file

When a canvas is serialized to a file
(e.g, by
[`toDataURL`](https://html.spec.whatwg.org/multipage/canvas.html#dom-canvas-todataurl) 
or
[`toBlob`](https://html.spec.whatwg.org/multipage/canvas.html#dom-canvas-toblob)
or
[`convertToBlob`](https://html.spec.whatwg.org/multipage/canvas.html#dom-offscreencanvas-converttoblob)),
the resulting encoded image shall preserve the behavior specified by `CanvasToneMapping`, if supported by the encoded image format.

Canvases that specify `"standard"` tone mapping shall be encoded in a way to maximize the precision in the SDR content.
If the canvas bitmap uses a floating-point pixel format and has a `PredefinedColorSpace` of `srgb-linear` or `display-p3-linear`,
but the encoding only supports fixed point pixel formats, then the encoding shall use the transfer function of `srgb` and `display-p3`.

Canvases that specify `"extended"` tone mapping shall be encoded using the HLG (for 8-bit) or PQ (for higher bit depth) transfer function.
If SMPTE ST 2094-50 HDR metadata can be attached to the resulting ICC profile, then that metadata shall indicate that the tone mapping to perform is a "clamp",
using the maximum linear HDR headroom value (with is 64 in that specification).

Support for encoding as AVIF and JpegXL should be considered for adding to the relevant APIs.

## Future directions

This explainer has only provided an API for indicating two tone mapping modes. We should consider adding the following modes:

```idl
  enum CanvasToneMappingMode {
      "standard",
      "extended",
      // The Reinhard tone mapping operator proposed in
      // Photographic tone reproduction for digital images.
      // https://doi.org/10.1145/566654.566575
      "reinhard",
      // The Reference White tone mapping operator proposed
      // in ISO 22028-5.
      "reference-white",
      // The tone mapping proposed in ITU-R BT.2408 Annex 5.
      "bt2408",
      // Tone mapping as specified by SMPTE ST 2094-50.
      "smpte-st-2094-application-5"
  };
```

Along with the following parameters.

```idl
  dictionary CanvasToneMapping {
    CanvasToneMappingMode mode = "standard";
    // Parameter indicating the HDR headroom of the content, which is used
    // by the "reinhard", "reference-white", and "bt2408" tone mapping
    // algorithms.
    optional double linearHDRHeadroom;
    // Parameters used by "smpte-st-2094-application-5" tone mapping algorithm.
    optional SmpteSt2094App5ColorVolumeTransform smpteSt2094App5ColorVolumeTransform;
  }; 

  dictionary SmpteSt2094App5ColorVolumeTransform {
    // SMPTE ST 2094-50 metadata set.
  }
```

The `"reinhard"`, `"reference-white"`, and `"bt2408"` curves can be encoded as
specific instances of SMPTE ST 2094-50 metadata (which allows them to be attached
to ICC profiles and videos).

They should be added to the specification along with SMPTE ST 2094-50, to
allow this simplication.

