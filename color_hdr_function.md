# CSS syntax for HDR colors parameterized by headroom

## Introduction

The proposal suggests a mechanism for allowing CSS colors to be displayed in HDR.

## Background

The [CSS color HDR specification](https://drafts.csswg.org/css-color-hdr/#predefined-HDR) provides some very useful additions to the roster of CSS color spaces.

## Motivation

The difficulty about designing a page that uses HDR colors is that there is a wide variety of different display capabilities.

For example, consider two HDR displays, one with HDR headroom of 2 and another with HDR headroom of 4.

The true HDR headroom of the display is not knowable, because that is a fingerprinting vector.

Suppose a page uses the colors `color(rec2100-linear 2 2 2)` and `color(rec2100-linear 3 3 3)`. On the display that has HDR headroom of 2, these colors will be the same. On the display that has HDR headroom of 3, these colors will be different. The author has no way of differentiating these two displays. Additionally, even if the HDR headroom were available, requiring a page to manually re-update its style whenever it changes would be burdensome.

## Related work

Gain map images, based on ISO 21496-1 are now nearly universally supported on mobile devices, and there is also significant support on laptop and desktop devices.

A gain map image implicitly represents an image at two different display HDR headrooms. Metadata in the image indicates the two headrooms involved. For each pixel, the pixel has one value at one headroom, and another value at another headroom. On displays that have headroom between the two, a well-defined interpolation is performed.

## Proposal

A similar scheme to gain map images could be done in CSS.

Consider the following `color-hdr()` function:

```css
  color-hdr() = color-hdr([ <color> && <number [1,infty]>? ]#{2})
```

This can be used to represent a color that behaves the same way that a gain map image pixel behaves.

Consider the color:

```css
  color-hdr(
      color(rec2100-linear 0.9 1.0 0.8) 1,
      color(rec2100-linear 1.8 2.0 1.5) 2);
```

On a display with HDR headroom <=1, this color will display as `color(rec2100-linear 0.9 1.0 0.8) 1`.

On a display with HDR headroom >=2, this color will display as `color(rec2100-linear 1.8 2.0 1.5) 2`.

For displays that have an hdr headroom H with 1 < H < 2, the color is interpolated in linear space, weighted by the log of the HDR headroom.

To interpolate between color c1 at headroom H1 and color c2 at headroom H2 when the target headroom is H.

* Let `w1 = clamp((log(H) - log(H2)) / (log(H1) - log(H2)), 0, 1)`
* Let `w2 = clamp((log(H) - log(H1)) / (log(H2) - log(H1)), 0, 1)`
* Note that `w2 = 1 - w1`
* Let `eps=0.001` (one linear just noticeable difference)
* The interpolated result is `pow(c1 + eps, w1) * pow(c2 + eps, w2) - eps`

In ISO 21496-1, the primaries of the space in which this interpolation is to be performed may be set by a parameter (often sRGB, P3, or Rec2020), and the value of `eps` is also a parameter. It may be worth adding those as parameters to be able to color match with gain map images.

## Gamut mapping interactions

Gamut mapping should be performed to the headroom of the target or the endpoint, whichever is lower. So in the above example, if `H=4` then we would still gamut map to `2=min(H, max(H1, H2))`.

For existing colors, their behavior can be treated as implicitly something like:

```css
  color-hdr(c, 1)`
```

By that mechanism, we can introduce in a backwards-and-forwards compatible way the convention that CSS colors are to be gamut mapped to the SDR gamut of the display, unless they explicitly opt-in to a potentially-higher-headroom gamut via the new syntax.

