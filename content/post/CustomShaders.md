---
title: "Running Custom WebGL Shaders Using Deck.gl for Solar Potential Analysis"
author: "| Atharva Garole"
date: 2025-02-10T17:08:07+05:30
draft: false
math: true
label.icon : imgs/logo.png
params:
  ShowToc: true
  TocOpen: false
  

cover:
   image: "imgs/image.jpg"
   alt: "Solar Potential Map"
   caption: "Gradient Solar Potential Map of New York City"
   relative: true
   hidden: false  

markup:
  highlight:
    codeFences: true
    guessSyntax: true
    lineNos: true
    style: "monokai"

---

## Introduction

Accurate solar potential analysis requires **precise shadow modeling**—a challenge when dealing with large-scale city models in real time. Traditional GIS tools often rely on precomputed static shadow maps, but for **dynamic sun positions and interactive analysis**, we need a more flexible approach.

Leveraging [**Deck.gl's**](https://www.deck.gl) **WebGL-powered rendering pipeline**, we implemented a **custom shader-based solution** to calculate **real-time solar exposure** on buildings. By extending Deck.gl’s built-in layers with custom [**GLSL shaders**](https://developer.mozilla.org/en-US/docs/Games/Techniques/3D_on_the_web/GLSL_Shaders), we were able to:

- Compute **Dynamic shadows** based on the sun’s position.  
- Use **Normal vector dot products** to determine light intensity per surface.  
- Modify **Fragment shaders** for accurate per-pixel lighting calculations.  
- Optimize **real-time rendering** for large-scale city models.  

By integrating **custom shader extensions**, we fine-tuned lighting interactions and implemented **shadow mapping** techniques, making our solar energy assessments both **scientifically accurate and computationally efficient**.       

## Advanced Real-Time Shadow Computation

When planning solar energy systems, shadows play a critical role in determining efficiency. Rooftops and building facades may seem ideal for solar panels, but if they are shaded for most of the day, energy output drops drastically.

To solve this, we built an interactive solar potential analysis tool using [**Deck.gl**](https://www.deck.gl) and WebGL shaders. The goal? Simulate shadows in real time, visualize solar exposure, and estimate Building Integrated Photovoltaic (BIPV) potential dynamically.

Let’s break this down!

### Sun Position & Dynamic Light Source

Shadows are entirely dependent on the sun’s azimuth and elevation, which change throughout the day. We compute these values dynamically and feed them into **Deck.gl’s SunLight API**, which acts as a directional light source:

```javascript
import { SunLight } from '@deck.gl/core';

const sunLight = new SunLight({
  timestamp: Date.now(), // Dynamic time-based sunlight
  color: [255, 255, 255],
  intensity: 1.0
});
```

This ensures that shadows shift in real time as the user adjusts the time slider, accurately reflecting sunlight exposure patterns.


### Shadow Map Generation

We start by creating a framebuffer object (FBO) to store depth and color information for our shadow map. The `ShadowPass` class extends `LayersPass` to create and manage this framebuffer.

```javascript
this.fbo = device.createFramebuffer({
  id: 'shadowmap',
  width: 1,
  height: 1,
  colorAttachments: [shadowMap],
  depthStencilAttachment: depthBuffer
});
```

- **Color attachment**: Contains the depth map as a texture.
- **Depth attachment**: Ensures depth testing works correctly.



### Rendering the Shadow Map

The `render` method resizes the framebuffer according to the viewport dimensions and clears it with a white color before rendering the shadow pass.

```javascript
const clearColor = [1, 1, 1, 1];
super.render({...params, clearColor, target, pass: 'shadow'});
```

- **Framebuffer resizing**: Ensures the shadow map’s resolution matches the viewport.
- **Depth test**: Controlled using `depthWriteEnabled` and `depthCompare` parameters.


### Depth Comparison in the Fragment Shader

The heart of the shadow projection technique lies in the GLSL fragment shader:

```glsl
uniform sampler2D shadowMap;
varying vec4 vShadowCoord;

void main() {
  float shadowDepth = texture2D(shadowMap, vShadowCoord.xy).z;
  if (vShadowCoord.z > shadowDepth) {
    gl_FragColor.rgb *= 0.5; // Render in shadow
  }
}
```

- **Shadow map sampling**: Fetches the depth value from the shadow map.
- **Depth comparison**: Determines if the fragment is in shadow.



### Shadow Bias Correction

A common issue in shadow mapping is **shadow acne**, where self-shadowing artifacts appear due to precision issues with depth comparison. We apply a small bias to the depth value to mitigate this:

```glsl
float bias = 0.005;
if (vShadowCoord.z - bias > shadowDepth) {
  gl_FragColor.rgb *= 0.5; // Render in shadow with bias correction
}
```

- **Bias tuning**: Must be adjusted based on the scene scale to avoid Peter Panning (shadows detaching from objects).


### Percentage Closer Filtering (PCF)

To achieve soft shadows, we implement **Percentage Closer Filtering (PCF)**, which averages multiple depth comparisons around the fragment’s shadow coordinate:

```glsl
float shadow = 0.0;
vec2 texelSize = 1.0 / textureSize(shadowMap, 0);
for (int x = -1; x <= 1; x++) {
  for (int y = -1; y <= 1; y++) {
    float pcfDepth = texture2D(shadowMap, vShadowCoord.xy + vec2(x, y) * texelSize).z;
    shadow += vShadowCoord.z > pcfDepth ? 0.3 : 1.0;
  }
}
shadow /= 9.0;
gl_FragColor.rgb *= shadow;
```

- **Texel size calculation**: Ensures smooth sampling over the shadow map.
- **Soft shadow effect**: Creates a more realistic penumbra.


### Optimizing Shadow Rendering

1. **Mipmapping**: Enabled for the shadow map texture to improve sampling quality.
2. **Clamp-to-edge wrapping**: Prevents artifacts at texture boundaries.
3. **Blending and depth testing**: Configured in `getLayerParameters` for accurate depth comparison.

```javascript
return {
  ...layer.props.parameters,
  blend: false,
  depthWriteEnabled: true,
  depthCompare: 'less-equal'
};
```
{{< figure align=center src="/imgs/gif.gif" alt="A rotating solar panel GIF" caption="" >}}

---

## Lighting in Deck.gl

Lighting in Deck.gl is fundamental for rendering realistic urban environments. By leveraging physically based lighting models and GPU-accelerated shading techniques, we ensure accurate simulation of sunlight exposure.

### Types of Lighting in Deck.gl

Deck.gl provides three primary lighting models:

- **AmbientLight** – Uniform, directionless light that prevents complete darkness.
- **DirectionalLight** – Simulates sunlight with parallel rays, enabling dynamic shadows.
- **PointLight** – Emits light in all directions (less relevant for solar analysis).

For our project, **DirectionalLight** is key to accurately representing sunlight throughout the day.

```javascript
const directionalLight = new DirectionalLight({
  color: [255, 255, 255],
  intensity: 1.0,
  direction: [-1, -1, -1]
});
```

### Shading & Material Effects

Deck.gl applies **Phong shading** to simulate how surfaces react to light. This involves:

- **Diffuse Lighting** – Reflects light based on surface angle to the light source.
- **Specular Highlights** – Adds realistic reflections for glass and metal.
- **Ambient Reflection** – Prevents pitch-black areas by simulating indirect light scattering.


{{< rawhtml >}}
<div style="position: relative; width: 100%; height: 450px; overflow: hidden; border-radius: 12px;">
  <iframe 
    style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; 
           border: none; display: block; transform: scale(1.01); transform-origin: center;" 
    scrolling="no" title="deck.gl LightingEffec" 
    src="https://codepen.io/vis-gl/embed/ZZwrZz?default-tab=result&editable=true&theme-id=dark" 
    frameborder="0" loading="lazy" allowtransparency="true" allowfullscreen="true">
  </iframe>
</div>
{{< /rawhtml >}}






---

## Surface Normals and Dot Product in Lighting Calculations

In Deck.gl and real-time rendering, the dot product between a surface normal and a light direction vector is fundamental to determining how light interacts with surfaces. This relationship is captured using:

$$ \vec{N} \cdot \vec{L} = |\vec{N}| |\vec{L}| \cos(\theta) $$
$$ I = \max(0, \vec{N} \cdot \vec{L}) $$


Where:  
- $\( I \)$ is the diffuse light intensity.  
- $\( \vec{N} \)$ is the normalized surface normal vector.  
- $\( \vec{L} \)$ is the normalized light direction vector.  



In Deck.gl shaders:

```glsl
vec3 normal = normalize(vNormal);
vec3 lightDir = normalize(uLightDirection);
float intensity = max(0.0, dot(normal, lightDir));
```

This intensity value adjusts shading dynamically, highlighting surfaces based on their orientation to the sun.

---

## Writing Custom Shaders with Deck.gl Extensions

Deck.gl provides a powerful extension system for injecting custom WebGL shaders into its rendering pipeline. This flexibility is critical for accurate solar potential analysis.

### Understanding Shader Hooks in Deck.gl

Deck.gl uses shader hooks to allow developers to modify the built-in rendering pipeline without rewriting the entire shader. A LayerExtension in Deck.gl provides a clean way to attach these hooks to existing layers. Let’s break down how this works!

### Creating a Custom Shader Extension with Hooks

In the example, the `HeatMapExtension` class extends `LayerExtension` and defines shader injections via the `getShaders` method:

```javascript
class HeatMapExtension extends deck.LayerExtension {
  getShaders() {
    return {
      inject: {
        "vs:#decl": `
          varying vec3 vNormal;
        `,
        "vs:#main-end": `
          vNormal = normalize(geometry.normal);
        `,
        "fs:#decl": `
          varying vec3 vNormal;
          uniform vec3 uLightDirection;
        `,
        "fs:DECKGL_FILTER_COLOR": `
          vec3 lightDir = normalize(uLightDirection);
          vec3 normal = normalize(vNormal);
          float dotProduct = max(dot(normal, lightDir), 0.0);
          vec3 colorValue = mix(vec3(0.5), vec3(1.0), dotProduct);
          color = vec4(colorValue, 1.0);
        `,
      },
    };
  }
}
```

### Explanation of Shader Hooks
- `vs:#decl`: Injects a varying variable declaration into the vertex shader to pass normal data to the fragment shader.
- `vs:#main-end`: Captures and normalizes the surface normal from the geometry after the vertex shader's main function.
- `fs:#decl`: Declares the varying normal vector and the uniform for the light direction in the fragment shader.
- `fs:DECKGL_FILTER_COLOR`: A Deck.gl filter hook that modifies the fragment color based on the dot product of the surface normal and light direction, simulating diffuse lighting.

### Passing Uniforms to the Extension

To ensure that the custom shader extension receives the correct light direction, uniforms need to be passed explicitly to the layer using the `getUniforms` function. This allows dynamic updates to lighting without modifying the extension code itself.

```javascript
const lightDirection = [1, 1, 1]; // Example light direction

const geoJsonLayer = new deck.GeoJsonLayer({
  id: 'geojson-layer',
  data: 'https://example.com/geojson',
  filled: true,
  extruded: true,
  getFillColor: [255, 255, 255],
  getElevation: f => f.properties.height || 0,
  pickable: true,
  shadowEnabled: true,
  extensions: [new HeatMapExtension()],
  parameters: { depthTest: true },
  getUniforms: () => ({
    uLightDirection: lightDirection,
  }),
});

new deck.DeckGL({
  container: 'container',
  initialViewState: {
    longitude: 72.5714,
    latitude: 23.0225,
    zoom: 12,
    pitch: 45,
    bearing: 0,
  },
  controller: true,
  layers: [geoJsonLayer],
});
```

By leveraging the `getUniforms` function, the light direction can be dynamically passed to the shader, enabling responsive and interactive visualizations without needing to hard-code these values in the shader code itself.


---
{{< rawhtml >}}
<video autoplay loop muted  playsinline style="width: 100%; border-radius: 12px; height: auto;">
  <source src="/imgs/Heatmap.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>
{{< /rawhtml >}}


## Conclusion

By customizing Deck.gl’s shader pipeline, we achieved real-time, GPU-accelerated shadow modeling for urban-scale solar analysis. The ability to dynamically compute light exposure, surface shading, and shadows ensures a precise evaluation of BIPV and rooftop solar potential.

This approach empowers researchers, urban planners, and policymakers to interactively explore solar energy potential, making informed decisions on optimal photovoltaic placements. As WebGL and GPU computing continue to evolve, such shader-based methodologies will become essential tools in the future of urban energy modeling and sustainability planning.




## Citation
```bibtex
@article{CustomShader,
  title   = "Running Custom WebGL Shaders Using Deck.gl for Solar Potential Analysis.",
  author  = "Atharva Garole",
  journal = "solar-vortex.github.io",
  year    = "2025",
  month   = "Feb",
  url     = "https://solar-vortex.github.io/Solar-Vortex"
}

