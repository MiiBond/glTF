# OpenPBR Conversion Reference for glTF Materials

## Status

Draft reference material. This is not an extension specification and does not define
normative glTF behavior. Mappings marked **TODO** or **VERIFY** require author and
working-group input before implementation.

## Scope and conventions

This document describes a conversion inventory in the same slab order as OpenPBR:
base, specular, transmission, coat, fuzz, sheen, emission, and geometry/normal.
OpenPBR names are shown in `code font`; glTF names are shown in `code font` where
possible. A conversion should preserve the rendered result, not merely copy
parameter names.

Color values and textures must be interpreted in the color space required by their
source specification. The following placeholder is intentionally not normative:

> **TODO:** Record the authoritative OpenPBR color-space, unit, default, and lobe
> conventions used by the target revision of the specification.

### Extension inventory

The likely glTF inputs are listed below. “WIP?” describes the state of this
repository/reference, not the status of an upstream extension.

| glTF feature | WIP? | Role in conversion |
|---|---|---|
| Core `pbrMetallicRoughness` | No | Base color, metalness, roughness, and alpha coverage |
| `KHR_materials_specular` | TODO/VERIFY | Specular weight and color controls |
| `KHR_materials_ior` | TODO/VERIFY | Surface index of refraction |
| `KHR_materials_transmission` | TODO/VERIFY | Transmission weight |
| `KHR_materials_volume` | TODO/VERIFY | Thickness, attenuation, and volume behavior |
| `KHR_materials_clearcoat` | TODO/VERIFY | Coat lobe |
| `KHR_materials_sheen` | TODO/VERIFY | Sheen lobe |
| `KHR_materials_emissive_strength` | TODO/VERIFY | Emission scale |
| `ADOBE_materials_thin_transparency` | No (repository document) | Thin-surface transmission input |
| `KHR_materials_anisotropy`, iridescence, dispersion, and related extensions | TODO/VERIFY | Additional OpenPBR controls with no settled mapping here |

## Base slab

### Differences between glTF and OpenPBR

glTF core exposes a metallic-roughness workflow (`baseColor`, `metallic`,
`roughness`). OpenPBR separates base-layer controls such as `base_weight`,
`base_color`, `diffuse_roughness`, and `metalness` (exact names and defaults:
**VERIFY**). OpenPBR may represent a broader material response than the core glTF
model.

### glTF extensions needed and whether each is work-in-progress

Core `pbrMetallicRoughness` is sufficient for the common case. `KHR_materials_pbrSpecularGlossiness`
may be an input for legacy assets, but its conversion is not lossless. **TODO:**
decide whether legacy extension input is in scope for the reference.

### Conversion glTF -> OpenPBR

Candidate starting point:

```text
base_color      <- pbrMetallicRoughness.baseColorFactor * baseColorTexture
metalness       <- pbrMetallicRoughness.metallicFactor * metallicTexture.B
roughness       <- pbrMetallicRoughness.roughnessFactor * metallicTexture.G
```

The effective base color for a metallic-roughness material is renderer-dependent
when alpha, vertex color, or extensions participate. **VERIFY** the exact OpenPBR
base-layer equations before implementing.

### Conversion OpenPBR -> glTF

Map the compatible base color, metalness, and roughness fields to
`pbrMetallicRoughness`. Bake or approximate controls with no glTF equivalent.
Preserve alpha coverage separately through `alphaMode`, `baseColorFactor.a`, and
the base-color texture alpha where applicable.

### Limitations and gotchas

The core model cannot carry every OpenPBR base control, and texture channel
packing differs between workflows. Do not assume that an OpenPBR texture can be
placed in the glTF metallic-roughness channels without a bake.

### What KHR_materials_openpbr could simplify or fix

It could define one material container with explicit base-layer fields, texture
color-space rules, defaults, and a loss-reporting mechanism instead of requiring
several extensions and renderer-specific reconstruction.

## Specular slab

### Differences between glTF and OpenPBR

OpenPBR exposes explicit specular controls (for example `specular_weight`,
`specular_color`, `specular_roughness`, and an IOR-related control). Core glTF
derives dielectric reflectance primarily from IOR defaults and uses roughness for
the shared microfacet lobe.

### glTF extensions needed and whether each is work-in-progress

`KHR_materials_specular` and `KHR_materials_ior` are likely required. In this
repository they are **TODO/VERIFY** because no corresponding specifications are
present.

### Conversion glTF -> OpenPBR

Use glTF IOR, specular weight, and specular color when available. Otherwise derive
the dielectric interface from the glTF default IOR and map the core roughness to
the compatible specular roughness. **VERIFY** whether OpenPBR's IOR level is
equivalent to a scalar F0 or must be derived with Schlick:

```text
F0 = ((ior - 1) / (ior + 1))^2
```

### Conversion OpenPBR -> glTF

Map IOR to `KHR_materials_ior` and specular controls to
`KHR_materials_specular` when their semantics match. If only core glTF is
available, bake compatible F0 behavior into the closest supported parameters and
record the approximation.

### Limitations and gotchas

F0, IOR, specular weight, and specular color are not interchangeable in all
material models. Clamping a colored or layered specular response to core glTF can
change energy balance.

### What KHR_materials_openpbr could simplify or fix

It could make IOR, F0, specular weight, and roughness relationships explicit,
including texture multiplication order and a defined fallback when a renderer
supports only core metallic-roughness.

## Transmission slab

### Differences between glTF and OpenPBR

OpenPBR separates transmission from surface reflection and can include color,
depth, scattering, and thin/thick behavior. Core glTF alpha coverage is not optical
transmission.

### glTF extensions needed and whether each is work-in-progress

`KHR_materials_transmission`, `KHR_materials_volume`, `KHR_materials_ior`, and
`ADOBE_materials_thin_transparency` may be needed. The first three are
**TODO/VERIFY** for this repository; the Adobe extension is documented here but
does not cover all volumetric OpenPBR behavior.

### Conversion glTF -> OpenPBR

Map transmission weight from `transmissionFactor * transmissionTexture` or the
Adobe `transmissionFactor * transmissionTexture`. Map IOR where available.
**TODO/VERIFY:** define how glTF thickness, attenuation color, attenuation
distance, and thin-walled semantics map to OpenPBR transmission fields.

### Conversion OpenPBR -> glTF

Use `KHR_materials_transmission` for transmission weight, `KHR_materials_ior` for
IOR, and `KHR_materials_volume` for supported volume data. Use the Adobe thin
transparency extension only when the OpenPBR material is demonstrably thin.
Otherwise bake, approximate, or report data loss.

### Limitations and gotchas

Alpha coverage controls visibility; it must not be substituted for transmission.
Real-time implementations may use approximations for refraction, filtering, and
ordering. Thickness and absorption are especially difficult to preserve in a
surface-only asset.

### What KHR_materials_openpbr could simplify or fix

It could unify thin-walled and volumetric transmission, define optical units and
defaults, and state whether transmission is applied before or after other lobes.

## Coat slab

### Differences between glTF and OpenPBR

OpenPBR has a dedicated clear protective coat lobe with its own weight, color,
roughness, IOR, and normal controls. Core glTF has no coat lobe.

### glTF extensions needed and whether each is work-in-progress

`KHR_materials_clearcoat` is required for a direct candidate mapping and is
**TODO/VERIFY** in this repository.

### Conversion glTF -> OpenPBR

If `KHR_materials_clearcoat` is present, copy its weight, roughness, tint, IOR,
and normal inputs where semantics agree. Otherwise set coat weight to zero.
**VERIFY** the exact relationship between clearcoat tint and OpenPBR coat color.

### Conversion OpenPBR -> glTF

Map compatible coat controls to `KHR_materials_clearcoat`. Bake unsupported
coat color, IOR, or normal behavior into textures or the base response only with
an explicit approximation report.

### Limitations and gotchas

A coat changes the energy available to the underlying base layer. Flattening it
into base color or roughness is view- and lighting-dependent.

### What KHR_materials_openpbr could simplify or fix

It could specify a single coat lobe, including its normal space, texture transforms,
energy interaction with the base layer, and fallback behavior.

## Fuzz slab

### Differences between glTF and OpenPBR

OpenPBR can describe a retroreflective or fiber-like fuzz lobe with weight, color,
and roughness. glTF has no core fuzz material control.

### glTF extensions needed and whether each is work-in-progress

No repository extension provides a direct mapping. A fuzz extension is **TODO**
and would be work-in-progress by definition.

### Conversion glTF -> OpenPBR

Set fuzz weight to zero unless an application-specific extension or baked look is
available. **TODO:** identify any accepted glTF representation for fuzz.

### Conversion OpenPBR -> glTF

Preserve fuzz only through a future extension or a renderer-specific bake.
Do not silently encode it as base color, sheen, or roughness.

### Limitations and gotchas

Fuzz is view-dependent and can require specialized sampling. A flattened texture
will not reproduce changes in view, light, or scale.

### What KHR_materials_openpbr could simplify or fix

It could reserve explicit fuzz fields and define a predictable approximation for
clients without a fuzz-capable renderer.

## Sheen slab

### Differences between glTF and OpenPBR

OpenPBR exposes a sheen lobe, commonly through weight, color, and roughness.
Core glTF has no sheen controls.

### glTF extensions needed and whether each is work-in-progress

`KHR_materials_sheen` is the likely direct input/output extension and is
**TODO/VERIFY** in this repository.

### Conversion glTF -> OpenPBR

Copy sheen weight, color, and roughness when the extension is present. Otherwise
use zero sheen. **VERIFY** whether glTF sheen color is interpreted as a direct
color or a tint of a wavelength-dependent model.

### Conversion OpenPBR -> glTF

Map compatible fields to `KHR_materials_sheen`. Bake unsupported spectral or
layering behavior only when an approximation is acceptable.

### Limitations and gotchas

Sheen is not a substitute for fuzz or coat. Channel packing, transfer functions,
and lobe normalization must be preserved.

### What KHR_materials_openpbr could simplify or fix

It could align sheen naming, defaults, texture encoding, and energy compensation
with the OpenPBR lobe.

## Emission slab

### Differences between glTF and OpenPBR

glTF provides `emissiveFactor` and `emissiveTexture`. OpenPBR can separate
emission color from luminance or strength and may define units more explicitly.

### glTF extensions needed and whether each is work-in-progress

`KHR_materials_emissive_strength` is likely required for values above the core
range and is **TODO/VERIFY** in this repository.

### Conversion glTF -> OpenPBR

Map emissive color and texture to OpenPBR emission color. Use strength 1 unless
an emissive-strength extension is present. **VERIFY** whether the OpenPBR
luminance field is photometric and therefore needs unit conversion:

```text
emission = emission_color * emission_luminance
```

### Conversion OpenPBR -> glTF

Use `emissiveTexture`/`emissiveFactor` for representable values and
`KHR_materials_emissive_strength` for a separate scale. **TODO:** define
HDR encoding and unit-preserving behavior.

### Limitations and gotchas

Emission is not necessarily a light source in a glTF renderer. Clamping or
color-space conversion can materially change bright emitters.

### What KHR_materials_openpbr could simplify or fix

It could define emission units, HDR ranges, texture encoding, and the relationship
between emissive color, luminance, and renderer light emission.

## Geometry and normal slabs

### Differences between glTF and OpenPBR

glTF stores geometry separately from material parameters and uses vertex normals,
optional tangents, and `normalTexture`. OpenPBR material descriptions can include
normal inputs for the base, coat, and other lobes, plus thin-walled or geometry
controls.

### glTF extensions needed and whether each is work-in-progress

Core glTF normal/tangent data is required. `KHR_materials_clearcoat` may carry a
coat normal; `KHR_materials_volume` may carry thin/volume state. Their status here
is **TODO/VERIFY**.

### Conversion glTF -> OpenPBR

Preserve vertex normals, tangents, UV sets, normal-map scale, and texture
transform. Convert tangent-space normal samples only after confirming handedness
and coordinate conventions. **VERIFY** whether OpenPBR's geometry normal,
shading normal, and coat normal are distinct inputs.

### Conversion OpenPBR -> glTF

Map the primary normal to `normalTexture` and tangent data to `TANGENT` where
available. Map coat normal to its extension field. Bake separate lobe normals
when glTF has no corresponding slot.

### Limitations and gotchas

Normal maps do not change geometric silhouettes or true ray-traced refraction.
Mirrored UVs, missing tangents, nonuniform scale, and differing normal-map
conventions can invert or distort the result.

### What KHR_materials_openpbr could simplify or fix

It could define normal-space conventions and provide explicit per-lobe normal
slots, while clearly separating geometric opacity, alpha coverage, and
thin-walled behavior.

## Other relevant slabs and controls

### Differences between glTF and OpenPBR

OpenPBR may expose anisotropy, iridescence, dispersion, subsurface or volume
controls beyond the slabs above. glTF represents these through separate
extensions, and this repository does not contain a complete OpenPBR mapping.

### glTF extensions needed and whether each is work-in-progress

Potential inputs include `KHR_materials_anisotropy`, `KHR_materials_iridescence`,
`KHR_materials_dispersion`, and `KHR_materials_volume`; each is **TODO/VERIFY**
for this reference.

### Conversion glTF -> OpenPBR

Copy only fields with verified semantic, unit, and texture-space equivalence.
Leave unsupported OpenPBR controls at specified defaults and emit a data-loss
record. **TODO:** define the canonical loss-report format.

### Conversion OpenPBR -> glTF

Prefer the corresponding glTF extension. Otherwise preserve source data in
application metadata only if a round trip is required; metadata is not a
rendering substitute.

### Limitations and gotchas

Extension combinations can interact through energy conservation, texture
packing, alpha handling, and renderer feature limits. A successful JSON
translation does not prove visual equivalence.

### What KHR_materials_openpbr could simplify or fix

It could provide a versioned superset container, explicit capability/fallback
declarations, and machine-readable loss reporting for both conversion directions.

## Open questions for author input

* **TODO:** Identify the exact OpenPBR specification revision and link.
* **TODO:** Confirm canonical slab/property names, defaults, units, and texture
  color spaces against that revision.
* **TODO:** Decide whether this reference targets only glTF 2.0 or future glTF
  material revisions as well.
* **TODO:** Define conformance examples and perceptual tolerances for round trips.
* **VERIFY:** Confirm every extension status against the Khronos extension registry
  before promoting any statement above to normative specification text.
