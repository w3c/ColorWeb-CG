# 2D Canvas tone mapping

## Introduction

This proposal suggests a mechanism to indicate that the bitmap of a `CanvasRenderingContext2D` or `OffscreenCanvasRenderingContext2D` is HDR, and that it is to be rendered as HDR.

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

The proposal is to replicate `GPUCanvasToneMapping` and `GPUCanvasToneMappingMode` in the HTML specification, renamed to `CanvasToneMapping` and `CanvasToneMappingMode`.

To the `CanvasRenderingContext2DSettings` dictionary, add the following entry:

```idl
  partial dictionary CanvasRenderingContext2DSettings {
    CanvasToneMapping toneMapping = {mode:"standard"};
  }
```

This indicates that the default behavior is to restrict color values to the standard dynamic range of the target display, which is the current behavior of all browsers.

Once this text is in the HTML specification, remove the text from the WebGPU specification, and change WebGPU to use `CanvasToneMapping`.
This is the same as how WebGPU uses `PredefinedColorSpace`.

## Future additions

### Nontrivial tone mapping modes

An obvious absence here is a tone mapping mode that does something non-trivial.
This is because standard tone mapping representations are still under standardization.
There are plans to add two additional modes:

 * An expressive custom tone mapping mode that is parameterized by the targeted HDR headroom (likely similar to the ICC adaptive gain curve [proposal](https://www.color.org/specification/ICC.1_Adaptive_Gain_Curve.pdf))
 * A default tone mapping parameterized by targeted HDR headroom

When the canvas is output, this tone mapping will be "taken into account". This means the tone mapping will:

 * be used when displaying the `<canvas>` (it will be tone mapped to the display's HDR headroom)
 * be used when tone mapping to create pixels is needed. E.g, to tone map to
    * [the specified 2D canvas HDR headroom](https://github.com/whatwg/html/issues/11165) when drawing using `drawImage`
    * [the specified WebGL HDR headroom](https://github.com/KhronosGroup/WebGL/issues/3735) when importing to WebGL
    * [the specified WebGPU HDR headroom](https://github.com/gpuweb/gpuweb/issues/5236) when importing to WebGPU
 * be included in the ICC profile when creating an image, so an output `<img>` crated from a `<canvas>` will look the same
 * be included in video metadata when creating a video, so a `<video>` created from the a `<canvas>` will look the same

This means that whatever tone mapping we add should be expressable in still images (likely carried via ICC) and in videos (likely carried ITU-T T.35).
There is work ongoing in this direction.

### Interactions with image and canvas output

Until the aformentioned future additions based on HDR headroom parameterized tone mapping, the value of `toneMapping` will not affect the results when exporting a canvas as an image or a video.

Because both tone mapping modes (`"standard"` and `"extended"`) do not actually do any (non-trivial) tone mapping, this is acceptable.

### Future prefined color spaces

At present, `PredefinedColorSpace` only includes `"srgb"` and `"display-p3"`.

Physically based rendering applications will certainly want [`"rec2100-linear"`](https://drafts.csswg.org/css-color-hdr/#valdef-color-rec2100-linear).

Of note is that `"rec2100-linear"` cannot generally be used for serializing to a bitmap (or video), and so we will need to update the details of [serializing](https://html.spec.whatwg.org/multipage/canvas.html#serialising-bitmaps-to-a-file) to indicate that a different color space (ITU-R BT.2100 PQ or HLG) must be used.
For lower bit depth encoding (e.g, JPEG), HLG encoding is preferable because it will result in less banding.
For higher bit depth encoding, PQ encoding is preferable because it can express a higher HDR headroom (5.6) than HLG (2.3).

It may be that [`"rec2100-pq"`](https://drafts.csswg.org/css-color-hdr/#valdef-color-rec2100-pq) and [`"rec2100-hlg"`](https://drafts.csswg.org/css-color-hdr/#valdef-color-rec2100-hlg) will also be desirable.

