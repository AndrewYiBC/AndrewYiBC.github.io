---
permalink: /photon_mapping/
title: "Volumetric Photon Mapping Project"
author_profile: true
redirect_from: 
  - /photon/
  - /photon_mapping/
  - /photon_mapping.html
---

In computer graphics, **caustics** refer to patterns of light formed when light rays focus onto specifc areas of a surface, often through specular reflections or refractions. When such light rays transmit through a participating medium, they can be scattered and result in **volume caustics**. In real life, caustics can commonly be observed under a wavy water surface or in the shadow of a curved glass of liquid. An example of volume caustics could be the sun casting beams of light through a misty morning forest -- an effect I witnessed during a visit to Sequoia National Park in November 2023, which also served as the inspiration for this project.

Drawn to the astonishing visual effects of caustics and volume caustics, I decided to implement **photon mapping** and **volumetric photon mapping**, and I was fortunate to have Professor [Theodore Kim](https://seas.yale.edu/faculty-research/faculty-directory/theodore-kim) supervise my project. This work builds upon the distributed ray tracer which I implemented from scratch in C++ for Professor Kim's Computer Graphics course project.

I implemented the algorithms based on a series of publications by Dr. Henrik Wann Jensen and Dr. Per H. Christensen [[1]](#1)[[2]](#2)[[3]](#3). In the sections following the demo gallery, I will provide a brief overview of the photon mapping and volumetric photon mapping algorithms, along with some technical details about my implementation.

## Gallery

| ![Water - Volumetric Photon Mapping](/images/photon_mapping/Water_VolumetricPhotonMapping_2048.png){:width="360" :height="360"} | ![Water - Photon Mapping](/images/photon_mapping/Water_PhotonMapping_2048.png){:width="360" :height="360"} |
|:--:|:--:|
| Box with Water Scene 1 - Volumetric Photon Mapping | Box with Water Scene 2 - Photon Mapping |

| ![Fog - Volume Caustics](/images/photon_mapping/Fog_VolumeCaustic_1024.png){:width="360" :height="360"} | ![Ring - Caustics](/images/photon_mapping/Ring_Caustic_2048.png){:width="360" :height="360"} |
|:--:|:--:|
| Volume Caustic through a Fog Medium | Cardioid Caustic of a Reflective Ring |

## Photon Mapping
The photon mapping algorithm was introduced by Dr. Henrik Wann Jensen in 1996 [[2]](#2). This overview synthesizes the key concepts I have learned from his seminal paper and comprehensive book on the subject [[1]](#1). Additionally, I will share some details about my implementation of the algorithms and the "box with water" demo scene, and discuss the challenges I encountered during the process.\
As a global illumination algorithm, photon mapping is capable of rendering indirect illumination and caustics more efficiently than pure Monte Carlo ray tracing methods. The technique can also be extended to volumetric photon mapping, which supports participating media and renders volume caustics.\
The standard photon mapping algorithm consists of two passes:

### 1st Pass: Photon Tracing

* **Photon Representation**\
Before diving into the photon tracing process, it is important to briefly describe the fundamental element of the algorithm -- the **photon**. In my implementation, I created a `Photon` class, where each photon instance stores its position, direction, and color (power, in RGB). The class also includes several boolean flags to track information such as wether the photon has been scattered, but I will omit the technical details for now.\
It is worth noting that, by including position and direction information, each photon can effectively be treated as a ray. This allows the ray tracing algorithm to be adapted for photon tracing with only slight modifications.

* **Photon Emission**\
Photon tracing is a **forward ray tracing** process, as opposed to **backward ray tracing**, which starts at the camera. In photon tracing, each photon is emitted from a light source, and its starting position and direction can be sampled based on the characteristics of the light source. For example, the starting position could be uniformly distributed across an area light source, and the starting direction could be sampled within a hemisphere or a cone. Once generated, the photon is treated as a ray and traced through the scene.\
In my box-with-water demo scene, I implemented several sampling strategies. However, in the final demo, the main light source on the ceiling is modeled as a cosine-weighted hemispherical point light source to produce sharper caustics.\
One additional detail is that if a light source emits \\(N\\) photons, then each photon should carry \\(1/N\\) of the light source's power.

* **Photon Scattering**\
When a photon hits the scene geometry (i.e. a ray intersection), it interacts with the surface and can either be reflected (diffusely or specularly), transmitted, or absorbed. The action of the photon is determined using a technique called **Russian roulette**, which eliminates the need to generate additional photons during the tracing process.\
For example, when a photon hits a refractive surface, instead of splitting its power into a transmitting photon and a reflecting photon based on the Fresnel equations, the Russian roulette algorithm will let the photon either transmit or reflect entirely, retaining its original power.\
The Russian roulette algorithm determines the photon's action using a probability distribution derived from the interacting surface's material parameters. For instance, when a photon interacts with a surface, the surface's diffuse coefficient \\(d\\) and specular coefficient \\(s\\) could be used to assign probabilities: the photon can have a probability of \\(d\\) to be diffusely reflected, \\(s\\) to be specularly reflected, and \\((1 - d - s)\\) to be absorbed. Similarly, when a photon hits a refractive surface, the coefficients computed from the Fresnel equations (which use the refractive index of the surface material) can be used as probabilities for specular reflection and transmission, respectively.\
After the photon action is determined, the photon's properties, such as position and direction, are updated following ray tracing practices.

* **Photon Storing**\
At each interaction of a photon with a non-specular surface, the information of the incoming photon is stored in the **photon map** -- a three-dimensional data structure -- at the position of interaction. The photon map is maintained separately from the scene geometry, ensuring the decoupling of illumination and geometry. The key operation for the photon map is locating the nearest neighboring photons at any given position, which is essential for the subsequent rendering pass of the photon mapping algorithm. This makes the kd-tree -- a multi-dimensional generalization of the binary search tree -- an ideal data structure for the photon map.\
A common optimization in photon mapping is to omit storing photons at their first non-specular interaction, as direct illumination can typically be computed by the ray tracing component of the algorithm. With this approach, the photon map is used exclusively for indirect illumination.

| ![Visualization - Photon Tracing](/images/photon_mapping/Vis_PhotonTracing_1024.png){:width="256" :height="256"} | ![Visualization - Photon Storing](/images/photon_mapping/Vis_PhotonStoring_1024.png){:width="256" :height="256"} |
|:--:|:--:|
| Photon Tracing Visualization | Photon Storing Visualization |

## References
<div style="font-size: 16px; padding-left: 56px; text-indent: -56px; display: inline-block;">
  <div>
    <a id="1">[1]</a> Henrik Wann Jensen. 2001. Realistic image synthesis using photon mapping. A. K. Peters, Ltd., USA.
  </div>
  <div>
    <a id="2">[2]</a> Henrik Wann Jensen. 1996. Global illumination using photon maps. In Proceedings of the eurographics workshop on Rendering techniques '96. Springer-Verlag, Berlin, Heidelberg, 21–30.
  </div>
  <div>
    <a id="3">[3]</a> Henrik Wann Jensen and Per H. Christensen. 1998. Efficient simulation of light transport in scenes with participating media using photon maps. In Proceedings of the 25th annual conference on Computer graphics and interactive techniques (SIGGRAPH '98). Association for Computing Machinery, New York, NY, USA, 311–320. https://doi.org/10.1145/280814.280925
  </div>
</div>
