# ImageData Floating Point Color Values

## Introduction

This proposal introduces the ability to use floating-point pixel formats with `ImageData`.

### Current capabilities

The `ImageData` interface currently has a member `data` of type `Uint8ClampedArray`.
The pixel format of the data is always `RGBA` with 8 bit per component unsigned normalized format.

### Use cases and motivation

High dynamic range and wide color gamut content content often require more than 8 bits per component to avoid banding artifacts.

Medical applications (e.g, radiography) demand higher than 8 bits per component resolution.

Modern high end displays are capable of displaying more than 8 bits per component.

### Related proposals and issues

[Github issue](https://github.com/whatwg/html/issues/10856)

2D canvas support [pull request](https://github.com/whatwg/html/pull/10951)

2D canvas support [github issue](https://github.com/whatwg/html/issues/8708)

## Proposed changes

### Interface changes

Create the new union type `ImageDataArray`.

```idl
  typedef (Uint8ClampedArray or Float16Array) ImageDataArray;
```

Create the new enum type `ImageDataPixelFormat`.

```idl
  enum ImageDataPixelFormat {
    "rgba8-unorm",
    "rgba16-float",
  };
```

To the `ImageDataSettings` dictionary add a `pixelFormat` member of type `ImageDataPixelFormat`, and have it default to `"rgba8-unorm"`.

```idl
  partial dictionary CanvasRenderingContext2DSettings {
    ImageDataPixelFormat pixelFormat;
  };
```

To the `ImageData` interface:
* add a `pixelFormat` member of type `ImageDataPixelFormat`
* change the type of `data` from `Uint8ClampedArray` to `ImageDataArray`
* change the constructor's `Uint8ClampedArray` argument to `ImageDataArray`

```idl
  partial interface ImageData {
    readonly attribute ImageDataPixelFormat pixelFormat;
    readonly attribute ImageDataArray data;
    constructor(ImageDataArray data, unsigned long sw, optional unsigned long sh,
                optional ImageDataSettings settings = {});
  }
```

### Algorithm changes

To the [algorithm to initialize an `ImageData`](https://html.spec.whatwg.org/multipage/canvas.html#initialize-an-imagedata-object), allow for the `source` parameter to be `Uint8ClampedArray` or `Float16Array`.
Ensure consistency between the optional `settings` parameter and the optional `source` parameter. In particular:

* If _source_ is given and is of type `Uint8ClampedArray`:
  * If _settings_ was given and _settings_`["pixelFormat"]` is not `"rgba8-unorm"`, then throw an `"InvalidStateError"` exception.
  * Otherwise, set `pixelFormat` to `"rgba8-unorm"`.
* If _source_ is given of type `Float16Array`:
  * If _settings_ was given and _settings_`["pixelFormat"]` is not `"rgba16-float"`, then throw an `"InvalidStateError"` exception.
  * Otherwise, set `pixelFormat` to `"rgba16-float"`.
* If _source_ is not given:
  * If _settings_ was given:
    * Set `pixelFormat` to _settings_`["pixelFormat"]`
  * If `pixelFormat` is `"rgba16-float"`, then set `data` to a `Float16Array`. Otherwise, set `data` to a `Uint8ClampedArray`

### Preservation of precision during use

#### `ImageBitmap`

When converting an `ImageData` with `pixelFormat` `"rgba16-float"` to an `ImageBitmap` via [`createImageBitmap`](https://html.spec.whatwg.org/multipage/imagebitmap-and-animations.html#dom-createimagebitmap), the resulting `ImageBitmap` must not lose any precision.

### WebGPU

In WebGPU, `ImageData` may be used as source of texture data, via [`GPUCopyExternalImageSource`](https://www.w3.org/TR/webgpu/#gpucopyexternalimagesourceinfo).
Copies using an `ImageData` with `pixelFormat` of `"rgba16-float"` must work, and not lose any precision.

### WebGL

In WebGL, `ImageData` may be used as source of texture data, via [`TexImageSource`](https://registry.khronos.org/webgl/specs/latest/1.0/#5.14).
Copies using an `ImageData` with `pixelFormat` of `"rgba16-float"` must work, and not lose any precision.

## Feature detection

The presence of this feature can be detected by checking for the presence of a `pixelFormat` attribute in `ImageData`.

## Remarks on choices

### Pixel format versus color type

This proposal uses the tern "pixel format", whereas the `CanvasRenderingContext2DSettings` interface uses "color type".
This is an intentional difference.

#### Motivation 1: The name is more semantically accurate

For `ImageData`, this truly is a pixel format that gives not just the data type (unsigned normalized 8-bit integer) but also the components and their layout (RGBA).
For 2D canvas, this is just the data type, and does not address the layout nor the color components present (RGB versus RGBA versus RGBX).

#### Motivation 2: Avoid the anti-pattern of using these types interchangeably

At present, the color types of a 2D canvas and the pixel formats of an `ImageData` have a clear 1:1 correspondance.
This may not remain the case.
If these two types were to have the same enum values, then one could imagine the code being written:

```js
  imageData = new ImageData(w, h, {pixelFormat:context2d.getContextAttributes().colorType});
```

This code is future-dangerous, because 2D canvas may get more color types than are supported in `ImageData`, whereupon this code would throw.

### Choice of no default parameter in `ImageDataSettings`

The `ImageDataSettings` dictionary does not specify a default `pixelFormat` value.

The default value is effectively `"rgba8-unorm"`, unless a `Float16Array` is specified.
Consider the following example:

```js
   let myFloat16Array = new Float16Array(4*w*h);
   let myImageData = new ImageData(myFloat16Array, w, h, {colorSpace:"rec2100-linear"});
```

If it were the case that `ImageDataSettings` set `pixelFormat` to `"rgba8-unorm"` by default, then this would throw an exception, because of the mismatch between the `data` type and the `pixelFormat` value.

This would be the case even if we set a default value for the `Float16Array` constructor

```idl
    constructor(Float16Array data, unsigned long sw, optional unsigned long sh,
                optional ImageDataSettings settings = {pixelFormat:"rgba16-float"});
```

