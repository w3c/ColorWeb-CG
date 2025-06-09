# Screen HDR headroom

## Introduction

This proposal introduces the ability to query the
[HDR headroom](https://www.w3.org/TR/css-color-hdr-1/#introducing-headroom).
of a given
[`screen`](https://www.w3.org/TR/cssom-view-1/#dom-window-screen).

## API Background

This is broken off from
[the proposal](https://github.com/w3c/ColorWeb-CG/blob/hdr_canvas_r2/hdr_html_canvas_element.md)
to add HDR imagery to the HTML canvas element.

This is necessary to allow applications to perform custom tone mapping,
e.g, using the
[`"extended"`](https://www.w3.org/TR/webgpu/#canvas-configuration)
tone mapping mode in WebGPU.

## Headroom background

The HDR headroom of a display is the log of the ratio between
the display's peak luminance and the display's HDR reference white luminance.

For many phones and laptops (and some higher end stand-alone displays),
the display's HDR reference white luminance is the same as the display's
brightness.

The display's brightness can change (often rapidly) based on the ambient
conditions.
As a true real-world example, changes in the cloud cover outside will change
the brightness of a display positioned near a window.

This has two important implications:

* Access to the HDR headroom in Javascript gives the page significant
 information about the user's environment, which can be used for fingerprinting. This informs our decision to gate this property behind a permission API.

* The HDR headroom property of a `screen` will change independently of, other `screen` properties (e.g, size, resolution, color volume, etc).

## Proposed changes

### `ScreenDetailed` IDL changes

To the [`ScreenDetailed`](https://www.w3.org/TR/window-management/#screendetailed) interface, add the following attributes:

```idl
  partial interface ScreenDetailed : Screen {
    readonly attribute float headroom;
    attribute EventHandler onheadroomchange;
  }
```

### Event firing

The `onheadroomchange` attribute is an
[event handler IDL attribute](https://html.spec.whatwg.org/multipage/webappapis.html#event-handler-idl-attributes)
whose [vent handler event type](https://html.spec.whatwg.org/multipage/webappapis.html#event-handler-event-type)
is `headroomchange`.

When the `headroom` of a `ScreenDetailed` _screenDetailed_ object changes,
[queue a global task](https://html.spec.whatwg.org/multipage/webappapis.html#queue-a-global-task)
on the
[relevant global object](https://html.spec.whatwg.org/multipage/webappapis.html#concept-relevant-global)
of _screenDetailed_ using the
[window placement task source](https://www.w3.org/TR/window-management/#window-placement-task-source)
to
[fire an event](https://dom.spec.whatwg.org/#concept-event-fire)
named `headroomchange` at _screenDetailed_.

## Issues

### Privacy concerns

The HDR headroom of a screen is a proxy for the ambient lighting in the user's environment.
This is potentially sensitive information, and can be used as a fingerprinting vector.

The HDR headroom is exposed only via the `ScreenDetailed` interface,
which is is a [default powerful feature](https://www.w3.org/TR/permissions/#dfn-default-powerful-feature)
identified by the name `"window-management"`.

Of note is that in [this issue](https://github.com/w3c/window-management/issues/148) is
proposing another feature to allow read-only access to `Screen` and `ScreenDetailed` information.
That issue is independent of, but in line with, the goals of this proposal.

### Other color information

[The proposal](https://github.com/w3c/ColorWeb-CG/blob/hdr_canvas_r2/hdr_html_canvas_element.md)
from which this is taken
also proposes exposing the color volume (also known as color gamut) of the display device.
This is not included in this proposal.
This section discusses how that would integrated, if added.

Unlike the HDR headroom of a display, the color volume of a display is not dynamic.
Therefore, the color volume would be treated as an ordinary
[advanced observable property](https://www.w3.org/TR/window-management/#screen-advanced-observable-properties)
of a `screen`, and changes in the color volume would use the same
[`onchange`](https://www.w3.org/TR/window-management/#ref-for-dom-screen-onchange%E2%91%A2) attribute and
[`change`](https://www.w3.org/TR/window-management/#eventdef-screen-change) event type
to notify of changes.

Some discussions of this feature have considered adding a separate "color information" structure.
Information about the color volume would be stored in a single structure, but that strucutre would be separate from the `headroom` attribute.

### Naming

This proposal uses the name `headroom` for the attribute and `headroomchanged` for the event type.

Alternatives include:

* `highDynamicRangeHeadroom` and `highdynamicrangeheadroomchanged`
* `hdrHeadroom` and `hdrheadroomchanged`

The current form is chosen for brevity and because there exists no other concept of headroom for which it could be confused.

