# HDR on the web, the big picture

This document provides and overview of
the many changes to many specifications
needed to provide a complete strategy for high dynamic range (HDR) on the web.

This is an elaboration of [this spreadsheet](https://docs.google.com/spreadsheets/d/1zq6vhz3w2aCp9sgCidpGncQxybaDyBBr9p3nVqASaUQ/edit?usp=sharing) tracking various HDR related features.
These specification changes affect many features covering many different areas, including:
CSS, 2D canvas, WebGL, WebGPU, and WebCodecs.

This document proposes to achieve an alignment about high level concept and directions (e.g, how to encode HDR headroom) across all APIs here in the [color on the web community group](https://www.w3.org/groups/cg/colorweb/).
From there, the details of the individual specification changes can be taken care of in their respective groups.

The following image shows an overview of the treatment of HDR content.
The numbered symbols indicate features that need to be added (except 0), and proposes an order for adding them.

There three broad components of the feature work:
* The "blue crosses" are for [displaying HDR elements](#displaying-html-elements-and-the-dynamic-range-limit-css-property)
* The "red stars" are for [drawing HDR content to a buffer](#drawing-hdr-content-to-a-bitmap-or-texture). See the individual explainers:
  * [2D canvas drawing HDR headroom parameter](canvas-compositing-headroom.md)
  * [WebGL texture import HDR headroom parameter](webgl-unpack-headroom.md)
  * [WebGPU texture import and copy HDR headroom parameter](webgpu-external-headroom.md)
* The "magenta suns" are for [attaching HDR metadata to canvases](#displaying-an-hdr-canvas). See the individual explainers:
  * [2D canvas tone mapping](canvas-tone-map.md)
  * [WebGL tone mapping](webgl-drawing-buffer-tone-map.md)
  * [SMPTE ST 2094-50 tone mapping](canvas-smpte-st-2094-50.md)

![Diagram of HDR big picture](hdr-big-picture.svg)

## Background

### HDR definition and not-definitions

HDR, on the web, means "can go brighter than the CSS color `white`".

HDR does not refer to high bit depth.
High bit depth is useful to HDR,
but it has uses independent of HDR.

HDR does not mean wide color gamut.
Wide color gamut can be useful to HDR,
but it has uses independent of HDR.

HDR on the web is not [Recommendation ITU-R BT.2100](https://www.itu.int/rec/r-rec-bt.2100) Hybrid-Log Gamma (HLG) or Perceptual Quantizer (PQ).
Those are common image and video encodings for HDR content, but are not the only way to express HDR content.

### HDR headroom

HDR headroom is defined as the ratio of peak luminance (of a thing) to the luminance of `white` (in that thing).

In all existing specifications that use this concept
([ISO 21496-1](https://www.iso.org/standard/86775.html) gain map images and [SMPTE ST 2094-50](https://github.com/SMPTE/st2094-50)),
it is defined as the log base 2 of that ratio.
This formulation can potentially be un-ergonomic and confusing because no APIs specify colors in the log2 domain, but many specify colors in a linear domain.

We will use the unambiguous term "linear HDR headroom" when an exact number is relevant to the discussion.
We will use the term "HDR headroom" to to the concept.
This is consistent with the tentative resolution of [issue #129](https://github.com/w3c/ColorWeb-CG/issues/129).

### HDR display definition and characterization

A display's HDR capability is parameterized entirely by its HDR headroom.

A display is HDR capable if its linear HDR headroom is greater than 1.
A display is not HDR capable, or is standard dynamic range (SDR), if its linear HDR headroom is equal to 1 (it can never be less than 1).

SDR displays are important and will be with us forever.
This strategy must be well-defined and high quality in SDR.
The personal choice to disable colors brighter than `white` will be with us forever.
Print is SDR (in this definition) and is forever.
E-ink is SDR (in this definition) and is forever.

### HDR image definition

An image is HDR if it specifies colors brighter than `white`.

This can be specified directly as pixel values (e.g, a PQ image),
or it can be specified by metadata (e.g, an ISO 21496-1 gain map or SMPTE ST 2094-50 gain curve).

### Tone mapping parameterized by targeted HDR headroom

Tone mapping is the process of transforming an image for display
at a specified targeted HDR headroom.
The only parameter for tone mapping is the targeted HDR headroom.
There are no other parameters.

One way to think about this is that an image that does not have tone mapping is a function that maps $x$ and $y$ coordinates to colors,
  so the color at $(x,y)$ is $f(x,y)$.
An image that has HDR tone mapping is a function that maps $x$ and $y$ coordinates, plus a targeted HDR headroom $H_\text{target}$ to colors,
  so the color at $(x,y)$ targeting HDR headroom $H_\text{target}$ is $f(x,y,H_\text{target})$.

All images that are HDR specify (or must have specified) how they are to be tone mapped to any targeted HDR headroom value.
This transformation is image-dependent and is specified by metadata.

Examples of specifications for metadata are:
* [ISO 21496-1 gain map](https://www.iso.org/standard/86775.html), which is a published standard
* [SMPTE ST 2094-50 gain curve](https://github.com/SMPTE/st2094-50), which is currently available for comment as a Public Committee Draft (PCD)
* A default treatment for images that contain no open standard metadata should be agreed upon, see [WhatWG issue #9112](https://github.com/whatwg/html/issues/9112)

An important property about these metadata is that they _prescribe_ an exact tone mapped color value for every pixel of the image or video, for every targeted headroom.
This is different from _descriptive_ metadata that describes the scene but leaves the transformation up to the implementation.
Metadata that is not prescriptive does not have behavior that can be used for interoperability testing, and is therefore incompatible with the web.

## Core features

### Displaying HTML elements, and the [`dynamic-range-limit`](https://www.w3.org/TR/css-color-hdr-1/#the-dynamic-range-limit-property) CSS property

All image and video elements are transformed to the HDR headroom of the window when they are displayed.

The user agent decides what the HDR headroom for a window is.
It may be the same as the HDR headroom of the current screen,
or it may some lower value, e.g, for background windows, for battery considerations, or based on user preferences.

The window's HDR headroom is a very high precision fingerprinting vector and is not to be exposed to via javascript.

The [`dynamic-range-limit`](https://www.w3.org/TR/css-color-hdr-1/#the-dynamic-range-limit-property) CSS property may be specified on elements to indicate that they should be further restricted.
This property applies to images, videos, canvases, and CSS colors.

### Drawing HDR content to a bitmap or texture

HDR images can be rendered an infinite number of ways, depending on the HDR headroom at which they are to be rendered.

There exists a general problem wherein an HDR image must be put into pixels in a buffer. At the moment the HDR image is put into pixels in a buffer, this infinite number of ways of representing it must be collapsed into a single representation at a specific HDR headroom.

This general problem has instances in 2D canvas, WebGL, and WebGPU.

* For 2D canvas, the [`drawImage`](https://html.spec.whatwg.org/multipage/canvas.html#canvasdrawimage) and similar functions exposed by the `CanvasDrawImage` interface included in `CanvasRenderingContext2D` and `OffscreenCanvasRenderingContext2D` perform this operation.
  * [This explainer](canvas-compositing-headroom.md) proposes adding a `globalLinearHDRHeadroom` attribute to the [`CanvasCompositing`](https://html.spec.whatwg.org/multipage/canvas.html#canvascompositing) interface.
* For WebGL, the `texImage2D` and related functions perform this operation.
  * [This explainer](webgl-unpack-headroom.md) proposes adding a `unpackLinearHDRHeadroom` attribute to the [`WebGLRenderingContextBase`](https://registry.khronos.org/webgl/specs/latest/1.0/#5.14) interface.
* For WebGPU, the [`copyExternalImageToTexture`](https://www.w3.org/TR/webgpu/#dom-gpuqueue-copyexternalimagetotexture) and [`importExternalTexture`](https://www.w3.org/TR/webgpu/#dom-gpudevice-importexternaltexture) functions perform this operation.
  * [This explainer](webgpu-external-headroom.md) proposes adding an `lienarHDRHeadroom` parameter to the [`GPUCopyExternalImageDestInfo`](https://www.w3.org/TR/webgpu/#gpucopyexternalimagedestinfo) and [`GPUExternalTextureDescriptor`](https://www.w3.org/TR/webgpu/#external-texture-creation) dictionaries.

The default behavior for all of these APIs is for them to tone map to SDR.
That way, any application that is oblivious to HDR will produce good results (as opposed to having out of range values clamped, etc).

None of these interfaces provide a way to get at the "raw pixels" of the image.

* An SDR image can be encoded using P3 pixel values versus Rec2020 pixel values, and that detail is not visible to these interfaces.
* An HDR image can be encoded using Display P3 as the pixel values (and a gain map or gain curve to go up to HDR) or as PQ as the pixel values (and a gain map or gain curve to go down to SDR), and that detail is not visible to these interfaces.

The valid range of values for this parameter is [1, `Infinity`].

### Displaying an HDR canvas

#### Trivial tone mapping

The default behavior for all HTML canvas elements is to limit their contents to the SDR color volume of the target display.

In WebGPU, a canvas can instead allow use of the full HDR color volume of the target display via the [`GPUCanvasToneMapping`](https://gpuweb.github.io/gpuweb/#dictdef-gpucanvastonemapping) structure.
In both of these cases, the transformation performed by tone mapping is the trivial tone mapping operation of "do nothing". The pixel values are unchanged regardless of the targeted headroom. See the next section for more involved tone mapping.
This behavior was limited to WebGPU because, at the time of writing, only WebGPU supported pixel formats that allowed expressing values outside of SDR color volumes. This has since been addressed for [2D contexts](https://github.com/whatwg/html/issues/8708) and for [WebGL](https://github.com/KhronosGroup/WebGL/pull/3222).

The WebGPU explainer [indiciated](https://github.com/ccameron-chromium/webgpu-hdr/blob/main/EXPLAINER.md#notes-on-canvasrenderingcontext2d-and-unification-of-apis) that this API should be generalized.
* [This 2D canvas explainer](canvas-tone-map.md) and [explainer PR](https://github.com/whatwg/html/pull/11734) move these parameters to the HTML specification, and allow the [`CanvasRenderingContext2DSettings`](https://html.spec.whatwg.org/multipage/canvas.html#canvasrenderingcontext2dsettings) to specify them.
* [This WebGL explainer](webgl-drawing-buffer-tone-map.md) indicates how these parameters would be added to WebGL, though it is somewhat out-of-date.

#### Using SMPTE ST 2094-50 metadata

The SMPTE ST 2094-50 specification defines non-trivial global tone mapping.

Once all APIs (2D, WebGL, and WebGPU) are use the same `CanvasToneMapping` interface described above, we can add support for tone mapping using SMPTE ST 2094-50 metadata to that interface, and it will be supported by all APIs.

[This explainer](canvas-smpte-st-2094-50.md) proposes such an interface.
SMPTE ST 2094-50 will be supported in videos and in images, and so exporting a canvas to an image or streaming it to a video will be supported.

#### Common tone mapping algorithms

It would be helpful to provide some common tone mapping algorithms. Examples could include:

* ITU-R BT.2408 Annex 5 tone mapper (will need some small adjustment because it is presented as an EETF)
* Reference white tone mapping operator (the default on macOS and iOS)
* Reinhard tone mapper

For the moment these have not been included in any explainer.
These can be implemented (or even specified) as specific sets of SMPTE ST 2094-50 metadata (and that way can be carried along with videos and images).

### CSS HDR Colors

The [hdr-color](https://drafts.csswg.org/css-color-hdr/#funcdef-hdr-color) functional allows specifying a color at several different HDR headrooms.
This allows an CSS color to color match a pixel in an HDR image at all HDR headrooms.

CSS colors using this functional will be affected by the `dynamic-range-limit` when rendering to the screen, and will be affected by the `globalHDRHeadroom` when rendering to a 2D canvas.

This specification also [defines HDR headroom](https://drafts.csswg.org/css-color-hdr/#hdr-headroom) using a log2 representation.
We should use the same convention (log2 versus linear) between CSS, 2D canvas, WebGL, and WebGPU.

### Detecting HDR capability

The [`dynamic-range`](https://www.w3.org/TR/mediaqueries-5/#dynamic-range) media query can report if the current screen is HDR or SDR.

Exactly how HDR the screen is, what its HDR headroom is, however, is not query-able.

## Example, how do I use the canvas APIs?

The question of "how would I use these APIs" often comes up.

If you doesn't want to do HDR, then you can just pretend these APIs don't exist. All canvas operations on all APIs will result in a canvas that renders content the same way it would appear on an SDR screen.

### Media carousel

Suppose you have an application that is just drawing a bunch of images or videos.

The first thing to do is to decide the "mastering HDR headroom" (this is like the Mastering Display Color Volume in SMPTE ST 2086). A good value for this would be 4x in linear space.

You'll set the `globalLinearHDRHeadroom` or `unpackLinearHDRHeadroom` or `linearHDRHeadroom` parameter to 4 when importing assets.
And, you'll just draw stuff as normal.

You'll configure your canvas to use SMPTE ST 2094-50 metadata indicating that the baseline HDR headrom is 4x (beware that in the parlance of that spec, everything is log2), and indicating the tone mapping curve to go to lower HDR headrooms (or just going with one of the pre-defined curves).

The user agent will then tone map to "whatever the actual HDR headroom of the display is".

### Physically based rendering

Suppose you have an application that is doing physically based rendering, where you're producing scene-referred luminance values in some buffer.

Once you have those scene-referred luminance values, you'll again decide what your "mastering HDR headroom" is.
Let's say you decide it's 4.

You will tone map your scene-referred luminance values to display-referred with linear HDR headroom 4, and write them in the backbuffer.
(Note that the mapping scene-referred values to display-referred values is called "tone mapping" and mapping display-referred values at headroom X to headroom Y is also called, confusingly, "tone mapping").

Maybe, if you arrange things right, you can manage to have this scene-referred to display-referred mapping be a no-op. But in general, physically-based scene-referred dynamic range is ginormous and totally unsuitable for direct display.

You'll then configure your canvas to use SMPTE ST 2094-50 metadata indicating how to tone map display-referred at 4x to other headroom values.

