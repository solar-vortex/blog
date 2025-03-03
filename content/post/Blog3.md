---
title: "Calculating BIPV Potential Using GHI, DNI, and Other Solar Parameters"
author: "| Sagar Singh"
date: 2025-02-10T17:08:07+05:30
draft: false
math: true
params:
  ShowToc: true
  TocOpen: false
  

cover:
   image: "imgs/bipv.gif"
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
Building-Integrated Photovoltaics (BIPV) present an innovative way to harness solar energy by integrating photovoltaic panels into a building’s **façades and rooftops**. Unlike conventional rooftop-mounted solar panels, BIPV panels serve both **architectural and functional purposes**, making them an essential part of sustainable urban development.

However, accurately estimating the energy generation potential of BIPV requires precise calculations that consider various **solar radiation parameters**, including:

- **Global Horizontal Irradiance (GHI)** – the total solar energy incident on a horizontal surface.
- **Direct Normal Irradiance (DNI)** – the amount of direct sunlight received perpendicular to the sun’s rays.
- **Diffuse Horizontal Irradiance (DHI)** – the scattered sunlight reaching the surface from the atmosphere.

In this blog, we discuss how these parameters contribute to **BIPV energy estimation** and how we integrate them into our **3D city model-based solar analysis** for an accurate and interactive assessment.



---

## 1. Understanding Key Solar Irradiance Parameters

### Global Horizontal Irradiance (GHI)
**GHI** is the total solar radiation received on a **horizontal surface** and is one of the most critical factors in solar energy analysis. It comprises two components:

$$\ GHI = DNI \cdot \cos(\theta_z) + DHI \$$

where:

- **DNI (Direct Normal Irradiance):** The direct sunlight component that reaches the Earth’s surface without being scattered.
- **DHI (Diffuse Horizontal Irradiance):** The portion of sunlight that is **scattered by molecules and aerosols** in the atmosphere.
- **θz (Zenith Angle):** The angle between the sun’s position and the vertical (the higher the sun, the lower the zenith angle).

For **flat rooftop-mounted PV panels**, **GHI** is generally sufficient for estimating solar potential. However, for **BIPV panels mounted on vertical walls**, **GHI alone is insufficient**, as they receive a significant portion of their energy from **DNI**.

{{< figure align=center src="/blog/imgs/GHI.png" alt="" caption="**Global Horizontal Irradiance**" >}}

### Why DNI is Important for BIPV?
Unlike horizontal rooftops, **building façades** are vertical structures that receive more sunlight when the sun is lower in the sky (morning and evening). This makes **DNI a more relevant factor** in estimating the energy generation of vertical BIPV panels.
{{< figure align=center src="/blog/imgs/DNI.png" alt="" caption="**Direct Normal Irradiance**" >}}

---

## 2. Calculating Solar Exposure for Vertical Surfaces

BIPV panels mounted on **vertical façades** experience different sunlight exposure patterns throughout the day. To **accurately estimate their energy generation**, we use the **Plane of Array (POA) irradiance formula**:

$$\ POA = DNI \cdot \cos(\theta) + DHI \cdot \frac{(1+\cos(\beta))}{2} + GHI \cdot \rho \cdot \frac{(1-\cos(\beta))}{2} \$$

where:

- **θ (Incidence Angle):** The angle between the sunlight and the panel’s surface.
- **β (Tilt Angle):** The panel's tilt from the horizontal plane (**90° for vertical BIPV**).
- **ρ (Albedo Coefficient):** The proportion of sunlight reflected from surrounding surfaces (ground, nearby buildings).

This calculation allows us to determine how much solar energy can be **captured by BIPV panels on different façades** and helps identify the most suitable orientations for **maximum energy generation**.

---

## 3. Dynamic Solar Irradiance Mapping for Enhanced BIPV Potential

### Why Traditional Solar Assessments Are Insufficient
Most **traditional solar potential studies** use **static assumptions** that do not consider real-world variations such as:

- **Daily and seasonal changes in the sun’s position**
- **Shading effects from nearby buildings**
- **Variations in atmospheric conditions**

To overcome these limitations, we implemented a **dynamic solar irradiance mapping system** that updates in real time based on various environmental factors.

### Key Components of Dynamic Mapping

#### Sun Position Calculations
- Using **solar azimuth** (horizontal angle) and **solar altitude** (vertical angle), we track how sunlight strikes different building surfaces **throughout the day and year**.
- This allows us to predict **peak solar exposure periods** for **façade-mounted BIPV panels**.

#### DNI & POA Adjustments
- Since **vertical surfaces** receive sunlight at different angles than rooftops, we **adjust the DNI and POA irradiance values** for each **building wall**.

#### Shading Analysis
- Shadows from **nearby structures** significantly **reduce BIPV efficiency**.
- Our system uses **3D shadow simulations** to determine which surfaces are **partially or completely shaded at different times of the day**.

#### Seasonal Variability Consideration
- Solar exposure **changes across seasons**, with lower angles in winter and higher angles in summer.
- Our model factors in **annual variations** to estimate the **long-term energy yield** for BIPV installations.

By integrating these dynamic components, our BIPV assessment becomes **far more accurate** than conventional **static solar potential maps**.

{{< figure align=center src="/blog/imgs/workflow.PNG" alt="" caption="" >}}

---

## 4. Integrating BIPV Analysis into a 3D City Model

We implemented our **solar analysis within an interactive 3D city model** using **Deck.gl and Three.js**, allowing users to visualize BIPV potential in a realistic urban environment.

### Step 1: Sun Position and Shadow Mapping
- We compute the **exact sun position** based on **date, time, and location**.
- A **shadow-casting algorithm** is applied to detect **self-shading and nearby structure obstructions**.

### Step 2: Energy Estimation for Each Building Face
- Each building façade is treated as a **potential BIPV surface**.
- The **POA irradiance formula** is used to calculate **daily and annual energy generation** for each orientation.

### Step 3: Data Visualization in 3D
- Solar exposure heatmaps are rendered on **building façades**.
- Users can **interact with the map**, click on buildings, and view **BIPV potential values** for each surface.
{{< figure align=center src="/blog/imgs/building.png" alt="" caption="" >}}

---

## 5. Using GDAL for Efficient Solar Data Processing

Handling large-scale solar data requires **efficient geospatial processing**. We use **GDAL (Geospatial Data Abstraction Library)** to:  

- **Read and extract** solar data from **GeoTIFF** files containing **GHI, DNI, and other irradiance datasets**.  
- **Reproject datasets** to match the coordinate system of the 3D city model.  
- **Perform pixel-based solar calculations** using NumPy integration with GDAL.  
- **Filter and analyze only relevant areas**, avoiding unnecessary computations.  

### **Using TIFF Files for Solar Energy Estimation**  
In our BIPV project, we used GDAL to read **DNI, GHI, and DHI TIFF files**, which store solar irradiance values for different geographical locations. The steps involved:  

1. **Loading the TIFF data**: Using GDAL’s `gdal.Open()` to read raster data.  
2. **Extracting pixel values**: Converting raster data into a NumPy array for calculations.  
3. **Reprojecting the data**: Aligning the TIFF coordinates with our 3D city model.  
4. **Calculating BIPV potential**: Using extracted solar parameters for POA calculations.  
5. **Visualizing results**: Mapping processed solar energy data onto **Deck.gl 3D models**.  

By using GDAL, we significantly improve the **efficiency** and **accuracy** of our BIPV potential calculations.  
{{< figure align=center src="/blog/imgs/gdal.png" alt="" caption="" >}}


---

## Conclusion

Building-Integrated Photovoltaics (BIPV) require **advanced solar energy estimation techniques** to determine their feasibility and efficiency. Our approach integrates:

- **Global Horizontal Irradiance (GHI) and Direct Normal Irradiance (DNI)** for precise calculations.
- **Plane of Array (POA) irradiance calculations** to assess vertical surfaces.
- **Dynamic sun position tracking and real-time shading analysis**.
- **3D visualization with Deck.gl and Three.js** for **interactive exploration**.
- **GDAL-based data processing** for efficient handling of **large-scale solar datasets**.

This **scientifically accurate** method ensures that urban planners, architects, and policymakers can make **data-driven decisions** for deploying **BIPV systems** efficiently. 




## Citation
```bibtex
@article{CustomShader,
  title   = "Calculating BIPV Potential Using GHI, DNI, and Other Solar Parameters",
  author  = "Sagar Singh",
  journal = "solar-vortex.github.io",
  year    = "2025",
  month   = "March",
  url     = "https://solar-vortex.github.io/blog/post/customshaders/"
}

