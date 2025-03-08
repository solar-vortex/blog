---
title: "3D Solar Rooftop Mapping with Google 3D Tiles, Deck.gl, and Three.js"
author: "Atharva Garole | Prathamesh Badgujar | Sagar Singh"
date: 2025-02-10T17:08:07+05:30
draft: false
math: true
weight : 1
params:
  ShowToc: true
  TocOpen: false
  

cover:
   image: "imgs/3d_solpan_optimize.gif"
   alt: "Solar Potential Map"
   caption: "Rendering 3D SolarPanels on Google Maps 3D Tiles"
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
Estimating solar potential isn’t just about mapping rooftops—it requires accurate panel placement, shading analysis, and real-time sun exposure calculations. To achieve this, we integrated Google’s **Photorealistic 3D Tiles** with **Deck.gl’s Tiles3DLayer** and **Three.js** for rendering dynamically placed solar panels.

While Deck.gl provides high-performance geospatial visualization, Three.js gives us **precise control** over materials, object transformations, and real-time shadows. Combining both frameworks allowed us to create an efficient, interactive 3D solar analysis tool.

---

## Loading Google 3D Tiles with Deck.gl’s Tiles3DLayer  
To render Google’s **Photorealistic 3D Tiles**, we used **Deck.gl’s Tiles3DLayer**, which efficiently loads 3D tilesets streamed from Google Maps. These tiles provide:
- True-to-scale building heights and orientations  
- Optimized streaming for large urban datasets  
- High-resolution rooftop textures for precise visualization  

### Implementation in Deck.gl  
We utilized **Tiles3DLayer** to load and visualize Google’s 3D Tiles in Deck.gl:

```js
import { Tile3DLayer } from '@deck.gl/geo-layers';

const tiles3DLayer = new Tile3DLayer({
  id: '3d-tiles',
  data: 'https://tile.googleapis.com/v1/3dtiles/root.json?key=YOUR_API_KEY',
  loader: Tiles3DLoader,
  onTilesetLoad: (tileset) => {
    console.log('Tileset loaded:', tileset);
  },
  pickable: true,
  pointSize: 2
});

new Deck({
  layers: [tiles3DLayer]
});
```
This efficiently streams 3D city models, ensuring smooth performance while loading urban datasets dynamically.

---

## Integrating Three.js and Deck.gl with Google Maps
While Deck.gl handles tile streaming, Three.js is used to **dynamically place solar panels** on rooftops. To align both rendering systems, we integrated Deck.gl’s camera system with Three.js using **GoogleMapsOverlay** and **Google Maps’ Three.js library**.

### Synchronizing Camera Systems  
To ensure seamless rendering, we synchronized the **Google Maps camera**, **Deck.gl’s Tiles3DLayer**, and **Three.js**:

```js
import { GoogleMapsOverlay } from '@deck.gl/google-maps';
import { WebGLRenderer, PerspectiveCamera } from 'three';

const map = new google.maps.Map(document.getElementById('map'), {
  center: { lat: 37.7749, lng: -122.4194 },
  zoom: 16,
  tilt: 45,
  mapId: 'YOUR_MAP_ID'
});

const overlay = new GoogleMapsOverlay({ layers: [tiles3DLayer] });
map.overlayMapTypes.push(overlay);

const threeRenderer = new WebGLRenderer();
const threeCamera = new PerspectiveCamera();
```
By syncing the **Google Maps camera** with Three.js, solar panels align perfectly with the real-world buildings in Google’s 3D Tiles.

{{< figure align=center src="/blog/imgs/flowchart.jpg" alt="" caption="" >}}

---

## Enumerating Solar Panel Orientation and Rendering  
Instead of procedurally placing panels, we used **Google's Solar API** to retrieve **panel locations and orientations**. We then passed this data to **Three.js’s InstancedMesh**, ensuring that panels align correctly with rooftops.

### Extracting Panel Orientation Data  
```js
let pitch = roofseg.pitchDegrees * Math.PI / 180;
let azimuth = roofseg.azimuthDegrees * Math.PI / 180;
```

### Applying Orientation to InstancedMesh  
```js
dummy.rotation.z = Math.PI - azimuth;
dummy.rotation.x = pitch * Math.cos(dummy.rotation.z);
dummy.rotation.y = pitch * Math.sin(dummy.rotation.z);
dummy.updateMatrix();
mesh.setMatrixAt(i, dummy.matrix);
```

To explain the mathematics behind this, we use two rotation matrices:

$$
R_z(\theta) = \begin{bmatrix}
\cos\theta & -\sin\theta & 0 \\\
\sin\theta & \cos\theta & 0 \\\
0 & 0 & 1
\end{bmatrix}
$$

$$
R_x(\phi) = \begin{bmatrix}
1 & 0 & 0 \\\
0 & \cos\phi & -\sin\phi \\\
0 & \sin\phi & \cos\phi
\end{bmatrix}
$$

Here, $\( R_z(\theta) \)$ rotates the panel about the vertical axis (adjusting for azimuth), while $\( R_x(\phi) \)$ rotates it about the horizontal axis to match the rooftop’s pitch. This combination ensures each solar panel is oriented correctly.

---

## Using InstancedMesh for Mass Rendering  

{{< rawhtml >}}
<video autoplay loop muted  playsinline style="width: 100%; border-radius: 12px; height: auto;">
  <source src="/blog/imgs/2d_solpan.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>
{{< /rawhtml >}}


Instead of creating separate mesh objects, we used **`InstancedMesh`** to render thousands of identical solar panels with a **single draw call**. Normally, rendering thousands of individual objects would lead to performance bottlenecks due to excessive draw calls. InstancedMesh allows us to batch-render multiple identical objects efficiently by sharing the same geometry and material, significantly improving performance.

```js
const panelGeometry = new THREE.BoxGeometry(1.50, 0.842, 0.1);
const panelMaterial = new THREE.MeshPhongMaterial({ color: 0x000080, shininess: 100 });
const panelMesh = new THREE.InstancedMesh(panelGeometry, panelMaterial, maxArrayPanelsCount);
overlay.scene.add(panelMesh);
```
- **`BoxGeometry`**(1.50, 0.842, 0.1) – Defines the size of the solar panel in meters.

- **`MeshPhongMaterial`** – Uses Phong shading to provide realistic lighting effects on the panels.

- **`InstancedMesh`** – Creates a single instance of the panel geometry but renders it multiple times efficiently.

- **`maxArrayPanelsCount`** – Specifies the maximum number of solar panels to be rendered in the scene.

- **`scene.add(panelMesh)`** – Adds the instanced mesh to the Three.js scene.

By using InstancedMesh, we significantly reduce GPU overhead and improve rendering speed, making large-scale urban solar mapping feasible in real time.

---

## Optimizing Tile Rendering with Tileset3D  

{{< rawhtml >}}
<video autoplay loop muted  playsinline style="width: 100%; border-radius: 12px; height: auto;">
  <source src="/blog/imgs/3d_solpan.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>
{{< /rawhtml >}}

To prevent unnecessary tile reloading and enhance rendering performance, we optimized **Deck.gl’s Tile3DLayer** settings. Without these optimizations, the application would waste resources by frequently unloading and reloading 3D tiles, leading to unnecessary network requests and reduced frame rates.

```js
tileset3d.options.unloadTiles = false;
tileset3d.options.refreshTiles = false;
tileset3d.options.preloadTiles = true;
tileset3d.options.viewDistanceScale = 3;
tileset3d.options.maximumMemoryUsage = 512;
tileset3d.options.refinementStrategy = 'REPLACE';
```
- **`unloadTiles`** = false – Keeps previously loaded tiles in memory instead of unloading them when they go out of view.

- **`refreshTiles`** = false – Prevents unnecessary tile reloading, improving rendering efficiency.

- **`preloadTiles`** = true – Loads tiles in advance, reducing latency when navigating the 3D scene.

- **`viewDistanceScale`** = 3 – Controls how far tiles should be loaded around the camera view, ensuring that tiles are available when needed without overloading memory.

- **`maximumMemoryUsage`** = 512 – Allocates up to 512MB of memory for tile caching, preventing excessive memory use while maintaining performance.

- **`refinementStrategy`** 'REPLACE' – Ensures that higher-detail tiles replace lower-detail tiles smoothly without rendering glitches.

---

## Conclusion  
By integrating **Google 3D Tiles, Deck.gl, and Three.js**, we created an efficient system for **visualizing rooftop solar potential**.
- **Deck.gl’s Tiles3DLayer** handled seamless 3D tile streaming.  
- **Three.js provided precise rendering** for dynamically placed solar panels.  
- **InstancedMesh and tile optimizations** ensured smooth performance.  

---


## Citation
```bibtex
@article{CustomShader,
  title   = "3D Solar Rooftop Mapping with Google 3D Tiles, Deck.gl, and Three.js",
  author  = "Atharva Garole | Prathamesh Badgujar | Sagar Singh",
  journal = "solar-vortex.github.io",
  year    = "2025",
  month   = "February",
  url     = "https://solar-vortex.github.io/blog/post/blog2/"
}
```


