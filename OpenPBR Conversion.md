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

## KHR_materials_openpbr
This is a special extension that marks a material as being authored as OpenPBR. That is, the intended interpretation of the material data is as an OpenPBR material. This has the effect of simplifying data conversion and/or allowing for compatibility. 

#### Simplification Example
Anisotropy is fully convertible between the two material models but a difference in roughness ramp requires a conversion that may involve baking textures with potentially different transforms. Using the `KHR_materials_openpbr` extension tells a loader to just interpret the anisotropy data as OpenPBR native so no complex conversion is necessary when importing into an OpenPBR-only renderer.

#### Compatibility Example
Specular Color behaves fundametally differently between OpenPBR and glTF in how it applies to both metals and glancing angles of dielectrics. There is no perfect conversion either way. Using the `KHR_materials_openpbr` extension is a way to signify that the intended interpretation of the specular color is as OpenPBR. This is useful if the renderer has a choice of material models to load the glTF into.


## Factors and Textures
In glTF, an individual material property is often split between factors and textures (e.g. `pbrMetallicRoughness.baseColorFactor` and `pbrMetallicRoughness.baseColorTexture`). These are generally multiplied together to get the final value. OpenPBR implicitly allows any property to be texturable and only defines a property's post-import form. To simplify conversions listed in this document, we will often refer to a combined glTF value relative to its OpenPBR equivalent (e.g. `pbrMetallicRoughness.baseColor == base_color`). It is assumed that the texture/factor combination must be handled.

## Base slab

glTF and OpenPBR both use a metallic-roughness workflow and are highly compatible in the base slab properties. glTF's core `pbrMetallicRoughness` exposes `baseColor`, `metallic`, and `roughness` while OpenPBR separates base color (albedo) into `base_weight` and `base_color` and distinguishes specular roughness from diffuse roughness (see details below).

### Base Weight and Color (i.e. Diffuse Albedo)

#### glTF -> OpenPBR

```text
base_weight     <- 1.0
base_color      <- pbrMetallicRoughness.baseColor
// When there is transmission, glTF's baseColor is used to tint the transmitted light at the surface.
// OpenPBR only has this capability for thin-walled materials or volumetric materials with no attenuation.
if (KHR_materials_transmission.transmission > 0) {
  // If there is no volume (or a volume with no attenuation), we can use OpenPBR's transmission_color for surface tinting.
  if (KHR_materials_volume.thickness == 0 || KHR_materials_volume.attenuation == white) {
    // The resulting material will be either thin-walled or will have transmission_depth == 0.
    // In both cases, transmission_color will be used to tint the transmission at the surface.
    transmission_color <- pbrMetallicRoughness.baseColor
  } else if (KHR_materials_openpbr exists) {
    // baseColor is assumed to be ignored for tranmitted light, just like in OpenPBR.
  } else {
    // The surface tinting from baseColor will need to be lifted into the coat slab (see below)
  }
}
```
##### Limitations:
Using `transmission_color` for surface-level tinting only works for thin-walled materials. If glTF includes `KHR_materials_volume.thickness > 0`, OpenPBR's `transmission_color` will be used for volumetric attenuation and the coat slab will need to be used to implement this tinting (see below).

#### OpenPBR -> glTF

```text
pbrMetallicRoughness.baseColor.rgb = base_color * base_weight
pbrMetallicRoughness.baseColor.a = geometry_opacity
if (transmission_weight > 0 || subsurface_weight > 0) {
  if (geometry_thin_walled || transmission_weight > 0 && transmission_depth == 0) {
    pbrMetallicRoughness.baseColor.rgb = mix(base_color * base_weight, transmission_color, transmission_weight)
  } else {
    // Material is volumetric with a non-white attenuation
    // Need to blend baseColor to white where the surface is transparent, unless we're using KHR_materials_openpbr
    if (not using KHR_materials_openpbr) {
      pbrMetallicRoughness.baseColor.rgb = mix(base_color * base_weight, white, transmission_weight)
    }
  }
}
```

##### Limitations:

glTF does not allow separate texture transforms between base color and opacity (i.e. alpha coverage) while OpenPBR does. Therefore, when baking OpenPBR's `base_color` and `geometry_opacity` to glTF, loss of data may occur. The same limitation exists for baking `base_weight` and `base_color` textures.

In thin-walled materials when `base_color` and `transmission_color` are different, the resulting glTF `baseColor` will be incorrect for intermediate values of `tranmission_weight` (i.e. not 0.0 or 1.0).

### Base Metalness

#### glTF -> OpenPBR
```text
base_metalness <- pbrMetallicRoughness.metallic
```

#### OpenPBR -> glTF
```text
pbrMetallicRoughness.metallic <- base_metalness
```

##### Limitations:
glTF does not allow separate texture transforms between specular roughness and metallic properties while OpenPBR does. Therefore, when baking OpenPBR's `base_metalness` and `specular_roughness` to glTF, loss of data may occur.

### Base Diffuse Roughness
This property requires the work-in-progress glTF extension, `KHR_materials_diffuse_roughness`.

#### glTF -> OpenPBR
```text
base_diffuse_roughness <- KHR_materials_diffuse_roughness.diffuseRoughness
```

#### OpenPBR -> glTF
```text
KHR_materials_diffuse_roughness.diffuseRoughness <- base_diffuse_roughness
```

## Specular slab

OpenPBR and glTF have largely compatible specular lobes but glTF requires some extensions for colored specular, non-default IOR, and specular anisotropy. However, specular color has some differences that need to be called out. In glTF, specular color is applied to dielectric materials at normal incidence (i.e. F0) and then transitions to white at glancing angles (i.e. F90). In OpenPBR, dielectrics have specular color applied to both F0 and F90. For metals (i.e. conductors), specular color is ignored in glTF while, in OpenPBR, it is used for the edge tinting using the F82 specular reflectance model. When a glTF contains the `KHR_materials_openpbr` extension, it is assumed that the asset was authored with the OpenPBR definition of specular color and should be interpreted as such.

### Specular Weight and Color
This property requires the glTF extension, `KHR_materials_specular`.

#### glTF -> OpenPBR
```text
specular_weight <- KHR_materials_specular.specular
if (KHR_materials_openpbr exists on this material) {
  specular_color <- KHR_materials_specular.specularColor
} else {
  specular_color <- mix(KHR_materials_specular.specularColor, white, pbrMetallicRoughness.metallic)
}
```

##### Limitations:
When a glTF does not use the `KHR_materials_openpbr` extension, it is not possible to reproduce the intended dielectric behaviour for F90 in OpenPBR. The best we can do is remove specular color for metals using a mix. Be aware that combining textures from glTF to produce OpenPBR's `specular_color` may result in a loss of data if different texture transforms for these properties are used.

#### OpenPBR -> glTF
```text
KHR_materials_specular.specularFactor <- specular_weight
KHR_materials_specular.specularColorFactor <- specular_color
```

### Specular Roughness

#### glTF -> OpenPBR
```text
specular_roughness <- pbrMetallicRoughness.roughness
```

#### OpenPBR -> glTF
```text
pbrMetallicRoughness.roughness <- specular_roughness
```

### Specular IOR
This property requires the glTF extension, `KHR_materials_ior` if IOR != 1.5.

#### glTF -> OpenPBR
```text
specular_ior <- KHR_materials_ior.ior || 1.5
```

#### OpenPBR -> glTF
```text
KHR_materials_ior.ior <- specular_ior
```

##### Limitations:
In glTF, IOR is not texturable. OpenPBR technically allows IOR to be texturable but the intent is for artists to use `specular_weight` to vary reflectivity instead. As such, this is unlikely to be a common issue for compatibility.

## Specular Anisotropy
This property requires the glTF extension, `KHR_materials_anisotropy`.

#### glTF -> OpenPBR
```text
anisotropy = KHR_materials_anisotropy.anisotropyStrength * KHR_materials_anisotropy.anisotropyTexture.b
if (KHR_materials_openpbr exists on this material) {
  specular_anisotropy_roughness <- anisotropy
} else {
  // Note that this conversion modifies both specular_roughness_anisotropy as well as specular_roughness
  baseAlpha = specular_roughness * specular_roughness;
  roughnessT = mix(baseAlpha, 1.0, anisotropy * anisotropy);
  roughnessB = baseAlpha;
  specular_roughness_anisotropy = 1.0 - roughnessB / max(roughnessT, 0.00001);
  specular_roughness = sqrt(roughnessT / sqrt(2.0 / (1.0 + (1.0 - specular_roughness_anisotropy) * (1.0 - specular_roughness_anisotropy))));
}
geometry_tangent = KHR_materials_anisotropy.anisotropyTexture.rg
```

##### Limitations:
Because KHR_materials_anisotropy.anisotropyTexture and pbrMetallicRoughness.metallicRoughnessTexture may have different texture transforms, converting to OpenPBR's `specular_roughness_anisotropy` and `specular_roughness` may result in a loss of data.

#### OpenPBR -> glTF
```text
if (using KHR_materials_openpbr) {
  KHR_materials_anisotropy.anisotropyTexture.b (or anisotropyStrength, if constant) <- specular_anisotropy_roughness
} else {
  // Convert to glTF-style anisotropy
  // Note that this conversion modifies both specular_roughness_anisotropy as well as specular_roughness
  baseAlpha = specular_roughness * specular_roughness;
  roughnessT = baseAlpha * Math.sqrt(2.0 / (1.0 + (1 - specular_roughness_anisotropy) * (1 - specular_roughness_anisotropy)));
  roughnessB = (1 - specular_roughness_anisotropy) * roughnessT;
  newBaseRoughness = Math.sqrt(roughnessB);
  newAnisotropyStrength = Math.min(Math.sqrt((roughnessT - baseAlpha) / Math.max(1.0 - baseAlpha, 0.0001)), 1.0);
  KHR_materials_anisotropy.anisotropyTexture.b (or anisotropyStrength, if constant) <- newAnisotropyStrength
  pbrMetallicRoughness <- newBaseRoughness
}
KHR_materials_anisotropy.anisotropyTexture.rg = geometry_tangent
```
##### Limitations:
Because anisotropyTexture can contain both strength (roughness) and rotation, having two different transforms between `specular_anisotropy_roughness` and `geometry_tangent` may result in a loss of data.

## Transmission slab

OpenPBR separates transmission from surface reflection and can include color, depth, scattering, and thin/thick behavior, while core glTF alpha coverage is not optical transmission.

### Transmission Weight
This property requires the work-in-progress glTF extension, `KHR_materials_transmission` (and, for thin-walled surfaces, `ADOBE_materials_thin_transparency`).

#### glTF -> OpenPBR
```text
transmission_weight <- KHR_materials_transmission.transmissionFactor
// If the surface is thin-walled
transmission_weight <- ADOBE_materials_thin_transparency.transmissionFactor
```

#### OpenPBR -> glTF
```text
KHR_materials_transmission.transmissionFactor <- transmission_weight
// If geometry_thin_walled == true
ADOBE_materials_thin_transparency.transmissionFactor <- transmission_weight
```

##### Limitations:
Use the Adobe thin transparency extension only when the OpenPBR material is demonstrably thin-walled. Otherwise bake, approximate, or report data loss. Alpha coverage controls visibility; it must not be substituted for transmission.

### Transmission Volume (Thickness and Attenuation)
This property requires the work-in-progress glTF extension, `KHR_materials_volume`. IOR is shared with the Specular slab's `IOR` property above.

#### glTF -> OpenPBR
```text
// TODO/VERIFY: define how glTF thickness, attenuation color, attenuation distance, and thin-walled semantics map to OpenPBR transmission fields
transmission_depth <- KHR_materials_volume.attenuationDistance
transmission_color <- KHR_materials_volume.attenuationColor
```

#### OpenPBR -> glTF
```text
KHR_materials_volume.attenuationDistance <- transmission_depth
KHR_materials_volume.attenuationColor <- transmission_color
```

##### Limitations:
Real-time implementations may use approximations for refraction, filtering, and ordering. Thickness and absorption are especially difficult to preserve in a surface-only asset.

## Coat slab

OpenPBR has a dedicated clear protective coat lobe with its own weight, color, roughness, IOR, and normal controls, while core glTF has no coat lobe.

### Coat Weight, Roughness, Tint, and IOR
This property requires the work-in-progress glTF extension, `KHR_materials_clearcoat`.

#### glTF -> OpenPBR
```text
coat_weight    <- KHR_materials_clearcoat.clearcoatFactor
coat_roughness <- KHR_materials_clearcoat.clearcoatRoughnessFactor
// VERIFY: the exact relationship between clearcoat tint and OpenPBR coat color
coat_color     <- KHR_materials_clearcoat.clearcoatTint
```

#### OpenPBR -> glTF
```text
KHR_materials_clearcoat.clearcoatFactor <- coat_weight
KHR_materials_clearcoat.clearcoatRoughnessFactor <- coat_roughness
KHR_materials_clearcoat.clearcoatTint <- coat_color
```

##### Limitations:
If `KHR_materials_clearcoat` is not present, set coat weight to zero. Bake unsupported coat color, IOR, or normal behavior into textures or the base response only with an explicit approximation report. A coat changes the energy available to the underlying base layer, so flattening it into base color or roughness is view- and lighting-dependent.

## Fuzz slab

OpenPBR can describe a retroreflective or fiber-like fuzz lobe with weight, color, and roughness, while glTF has no core fuzz material control.

### Fuzz Weight, Color, and Roughness
No glTF extension currently provides a direct mapping for this property; a fuzz extension is **TODO** and would be work-in-progress by definition.

#### glTF -> OpenPBR
```text
// TODO: identify any accepted glTF representation for fuzz
fuzz_weight <- 0.0
```

#### OpenPBR -> glTF
```text
// Preserve fuzz only through a future extension or a renderer-specific bake
// Do not silently encode it as base color, sheen, or roughness
```

##### Limitations:
Fuzz is view-dependent and can require specialized sampling. A flattened texture will not reproduce changes in view, light, or scale.

## Sheen slab

OpenPBR exposes a sheen lobe, commonly through weight, color, and roughness, while core glTF has no sheen controls.

### Sheen Weight, Color, and Roughness
This property requires the work-in-progress glTF extension, `KHR_materials_sheen`.

#### glTF -> OpenPBR
```text
// VERIFY: whether glTF sheen color is interpreted as a direct color or a tint of a wavelength-dependent model
sheen_weight    <- KHR_materials_sheen.sheenColorFactor
sheen_color     <- KHR_materials_sheen.sheenColorFactor
sheen_roughness <- KHR_materials_sheen.sheenRoughnessFactor
```

#### OpenPBR -> glTF
```text
KHR_materials_sheen.sheenColorFactor <- sheen_color * sheen_weight
KHR_materials_sheen.sheenRoughnessFactor <- sheen_roughness
```

##### Limitations:
If `KHR_materials_sheen` is not present, use zero sheen. Sheen is not a substitute for fuzz or coat. Bake unsupported spectral or layering behavior only when an approximation is acceptable, and preserve channel packing, transfer functions, and lobe normalization.

## Emission slab

glTF provides `emissiveFactor` and `emissiveTexture` while OpenPBR can separate emission color from luminance or strength and may define units more explicitly.

### Emission Color

#### glTF -> OpenPBR
```text
// VERIFY: whether the OpenPBR luminance field is photometric and therefore needs unit conversion
emission_color <- emissiveFactor * emissiveTexture
emission = emission_color * emission_luminance
```

#### OpenPBR -> glTF
```text
emissiveFactor <- emission_color
```

##### Limitations:
Emission is not necessarily a light source in a glTF renderer. Clamping or color-space conversion can materially change bright emitters.

### Emission Strength (Luminance)
This property requires the work-in-progress glTF extension, `KHR_materials_emissive_strength`.

#### glTF -> OpenPBR
```text
emission_luminance <- 1.0
// If KHR_materials_emissive_strength is present
emission_luminance <- KHR_materials_emissive_strength.emissiveStrength
```

#### OpenPBR -> glTF
```text
KHR_materials_emissive_strength.emissiveStrength <- emission_luminance
```

##### Limitations:
**TODO:** define HDR encoding and unit-preserving behavior for values above the core glTF representable range.

## Geometry and normal slabs

glTF stores geometry separately from material parameters and uses vertex normals, optional tangents, and `normalTexture`, while OpenPBR material descriptions can include normal inputs for the base, coat, and other lobes, plus thin-walled or geometry controls.

### Normals and Tangents

#### glTF -> OpenPBR
```text
// VERIFY: whether OpenPBR's geometry normal, shading normal, and coat normal are distinct inputs
geometry_normal <- NORMAL, TANGENT, normalTexture
```

#### OpenPBR -> glTF
```text
normalTexture <- geometry_normal
TANGENT <- geometry_tangent
```

##### Limitations:
Preserve vertex normals, tangents, UV sets, normal-map scale, and texture transform. Convert tangent-space normal samples only after confirming handedness and coordinate conventions. Normal maps do not change geometric silhouettes or true ray-traced refraction. Mirrored UVs, missing tangents, nonuniform scale, and differing normal-map conventions can invert or distort the result.

### Coat Normal
This property requires the work-in-progress glTF extension, `KHR_materials_clearcoat`.

#### glTF -> OpenPBR
```text
coat_normal <- KHR_materials_clearcoat.clearcoatNormalTexture
```

#### OpenPBR -> glTF
```text
KHR_materials_clearcoat.clearcoatNormalTexture <- coat_normal
```

##### Limitations:
Bake separate lobe normals when glTF has no corresponding slot.

## Other relevant slabs and controls

OpenPBR may expose anisotropy, iridescence, dispersion, subsurface or volume controls beyond the slabs above, while glTF represents these through separate extensions, and this repository does not contain a complete OpenPBR mapping.

### Additional OpenPBR Controls
This property may require the work-in-progress glTF extensions `KHR_materials_anisotropy`, `KHR_materials_iridescence`, `KHR_materials_dispersion`, and `KHR_materials_volume`.

#### glTF -> OpenPBR
```text
// Copy only fields with verified semantic, unit, and texture-space equivalence
// TODO: define the canonical loss-report format
```

#### OpenPBR -> glTF
```text
// Prefer the corresponding glTF extension
// Otherwise preserve source data in application metadata only if a round trip is required; metadata is not a rendering substitute
```

##### Limitations:
Leave unsupported OpenPBR controls at specified defaults and emit a data-loss record. Extension combinations can interact through energy conservation, texture packing, alpha handling, and renderer feature limits. A successful JSON translation does not prove visual equivalence.

## Open questions for author input

* **TODO:** Identify the exact OpenPBR specification revision and link.
* **TODO:** Confirm canonical slab/property names, defaults, units, and texture
  color spaces against that revision.
* **TODO:** Decide whether this reference targets only glTF 2.0 or future glTF
  material revisions as well.
* **TODO:** Define conformance examples and perceptual tolerances for round trips.
* **VERIFY:** Confirm every extension status against the Khronos extension registry
  before promoting any statement above to normative specification text.
