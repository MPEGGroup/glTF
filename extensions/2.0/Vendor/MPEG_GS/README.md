<!--
Copyright 2025 ISO/IEC JTC 1/SC 29/WG 7 (MPEG)
SPDX-License-Identifier: CC-BY-4.0
-->

# MPEG_GS

## Status

Draft

## Dependencies

Written against the glTF 2.0 specification.

This extension is designed to interoperate with MPEG timed media mechanisms for dynamic sequences, including timed accessors and circular buffers, as defined by MPEG-I Scene Description.

## Overview

Gaussian Splats provide an efficient representation for static and dynamic 3D scenes by compactly encoding local geometry and view dependent appearance.

This extension defines how 3D Gaussian Splats and 4D Gaussian Splat sequences are stored in an MPEG-I Scene Description and bound to glTF primitives for interoperability, backward compatibility, and progressive rendering.

A receiver that does not implement this extension can still render a baseline POINTS preview using standard glTF attributes POSITION and COLOR_0. In that fallback, COLOR_0.rgb carries the DC term for color, and COLOR_0.a carries opacity.

## Adding Gaussian Splats to primitives

Gaussian Splats shall be attached to a glTF primitive whose `mode` is `0` (POINTS). The primitive shall contain:

- Baseline attributes for fallback rendering: POSITION and COLOR_0.
- An `extensions.MPEG_GS` object that references the additional Gaussian Splat data.

### Baseline mapping

These properties map to existing glTF primitive attributes to ensure a baseline POINTS visualization.

| GS property | glTF attribute | Notes |
| --- | --- | --- |
| x, y, z | POSITION | VEC3 |
| f_dc_[0-2] | COLOR_0.rgb | May require normalization depending on encoding |
| opacity | COLOR_0.a | Alpha channel of COLOR_0 |

### Extension semantics

All GS specific attributes listed below shall be carried within `extensions.MPEG_GS` as references to a glTF accessor (static) or a timed accessor (dynamic). Unless explicitly stated otherwise, components are 32 bit IEEE 754 floating point values and little endian in buffers.

| Name | Type | Description |
| --- | --- | --- |
| orientation | VEC4 | Quaternion rotation (x, y, z, w) defining the local axes orientation of each splat |
| scale | VEC3 | Ellipsoid scale along local X, Y, Z axes defining the size of each splat |
| shCoeffR | SCALAR | Spherical harmonic coefficients for the red channel |
| shCoeffG | SCALAR | Spherical harmonic coefficients for the green channel |
| shCoeffB | SCALAR | Spherical harmonic coefficients for the blue channel |
| shCoeffFirst | SCALAR | Progressive layout first order coefficients, 9 coefficients per splat |
| shCoeffSecond | SCALAR | Progressive layout second order coefficients, 15 coefficients per splat |
| shCoeffThird | SCALAR | Progressive layout third order coefficients, 21 coefficients per splat |

### Spherical harmonic layouts

Two layouts are supported. Receivers shall support both layouts and shall follow the signaled layout.

- Progressive layout
  - The extension property `shLayout` shall be `"progressive"`.
  - `shCoeffFirst`, `shCoeffSecond`, and `shCoeffThird` shall contain 9, 15, and 21 coefficients per splat respectively.
  - The DC term (order 0) shall not be stored in these arrays and shall be reconstructed from COLOR_0.rgb.

- Per channel layout
  - The extension property `shLayout` shall be `"perChannel"`.
  - `shCoeffR`, `shCoeffG`, and `shCoeffB` shall each contain 15 coefficients per splat for that channel.
  - The DC term shall be reconstructed from COLOR_0.rgb.

Coefficient packing is linear. For an accessor with N splats and K coefficients per splat, the accessor count should be N times K, and coefficient i for splat s is at index (s times K) plus i.

## Processing model

### Static Gaussian Splats

For static Gaussian Splats, the receiver shall fetch the baseline attributes POSITION and COLOR_0 and the extension referenced attributes in `extensions.MPEG_GS` once and shall use them for rendering throughout the session.

### Dynamic Gaussian Splats (4DGS)

For dynamic Gaussian Splats, the extension may reference timed accessors. The sender announces temporal data using MPEG timed media mechanisms such as MPEG_media, MPEG_buffer_circular, and MPEG_accessor_timed. The receiver obtains per frame data accordingly and shall use a circular buffer to retrieve the latest available frame.

During each rendering cycle, the receiver retrieves updated attributes from the circular buffer pointing to the latest Gaussian Splat frame. If no new per frame data is available, the receiver shall continue to use the last available set of attributes.

### Progressive download and rendering

Progressive download is supported by delivering additional bufferViews and by signaling one of the spherical harmonic layouts. Upon initialization, the renderer fetches POSITION and COLOR_0 for a coarse representation. As more data becomes available, the renderer incrementally updates the splats by loading orientation, scale, and progressively refined spherical harmonic coefficients. Each incremental bufferView or frame improves fidelity without re decoding prior data.

## Example

This example shows a POINTS primitive with baseline fallback data and GS specific data referenced via the extension.

```json
{
  "meshes": [
    {
      "primitives": [
        {
          "mode": 0,
          "attributes": {
            "POSITION": 0,
            "COLOR_0": 1
          },
          "extensions": {
            "MPEG_GS": {
              "shLayout": "progressive",
              "orientation": 2,
              "scale": 3,
              "shCoeffFirst": 4,
              "shCoeffSecond": 5,
              "shCoeffThird": 6
            }
          }
        }
      ]
    }
  ]
}
```

## Schema

- [Primitive extension JSON Schema](./MPEG_GS.schema.json)
