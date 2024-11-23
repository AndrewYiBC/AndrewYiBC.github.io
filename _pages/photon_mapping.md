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
The photon mapping algorithm was introduced by Dr. Henrik Wann Jensen in his 1996 paper, "Global Illumination using Photon Maps" [[2]](#2). This overview synthesizes the key concepts I have learned from his seminal paper and his comprehensive book, *Realistic Image Synthesis Using Photon Mapping* [[1]](#1). Additionally, I will share some details about my implementation of the algorithms and the "box with water" demo scene, and discuss the challenges I encountered during the process.\
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
At each interaction of a photon with a non-specular surface, the information of the incoming photon is stored in the **photon map** -- a three-dimensional data structure -- at the position of interaction. This photon map is maintained independently from the scene geometry, ensuring the decoupling of illumination and geometry. The key operation for the photon map is locating the nearest neighboring photons at any given position, which is essential for the subsequent rendering pass of the photon mapping algorithm. This makes the kd-tree -- a multi-dimensional generalization of the binary search tree -- an ideal data structure for the photon map. During the photon tracing pass, photons are initially stored in a flattened array, which will be transformed into a kd-tree at the end of the process.\
A common optimization in photon mapping is to omit storing photons at their first non-specular interaction, as direct illumination can typically be computed by the ray tracing component of the algorithm. With this approach, the photon map is used exclusively for indirect illumination, which is the strategy I adopted in my implementation.

| ![Visualization - Photon Tracing](/images/photon_mapping/Vis_PhotonTracing_1024.png){:width="256" :height="256"} | ![Visualization - Photon Storing](/images/photon_mapping/Vis_PhotonStoring_1024.png){:width="256" :height="256"} |
|:--:|:--:|
| Photon Tracing Visualization | Photon Storing Visualization |

### 2nd Pass: Rendering

* **Radiance Estimate**\
During the rendering pass, the photon map is used to compute the reflected radiance at any given point in the scene. Without participating media, these points are always on the surfaces of the scene geometry.\
To estimate the reflected radiance at a specific position $$x$$, the nearest $$n$$ photons are retrieved by querying the photon map. The reflected radiance is then approximated using the following equation:\
------------------------------------------------------------------------------------------------------------
\$$
L_r(x,\vec{\omega}) \approx \sum_{p=1}^n f_r(x,\vec{\omega}_p, \vec{\omega}) \frac{\Delta\Phi_p(x,\vec{\omega}_p)}{\Delta A}
\$$
This is Equation (7.4) of [[1]](#1), where\
$$L_r$$ is the reflected radiance\
$$f_r$$ is the BRDF at $$x$$\
$$p$$ is a photon with power $$\Delta\Phi_p(x,\vec{\omega}_p)$$\
$$\Delta A$$ is the area ($$\Delta A = \pi r^2$$ assuming the photons are on the same surface)\
------------------------------------------------------------------------------------------------------------\
I will omit the detailed deriviation starting from the rendering equation in this overview. If you are interested, please refer to the source for more information.\
The canonical way to perform radiance estimation is to locate the nearest $$n$$ photons around $$x$$ and determine the radius $$r$$ based on the distance to the furthest photon (nearest neighbors search). However, I observed that with a sufficiently high photon count, radiance estimation produces better results when the search radius $$r$$ is fixed to a small value, allowing $$n$$ to vary with photon density (radius search). I found this approach particularly effective for rendering caustics, as it results in sharper edges.\
Another technique for rendering caustics involves splitting the photon map into a **global photon map** and a **caustics photon map**, with the latter dedicated to storing photons that have undergone specular reflections or transmissions. When I tested this approach, I used the nearest neighbors search for the global photon map and the radius search for the caustics photon map. However, this method did not result in a significant improvement in caustics quality while increasing both computational and memory costs. Therefore, the demo scenes use only a global photon map and the radius search.

* **The Cone Filter**\
The cone filter is a simple yet efficient technique for sharpening the rendered caustics. It assigns a weight of $$w_{pc}=(1-\frac{d_p}{kr})$$ to each photon $$p$$, where $$d_p$$ is the distance between $$x$$ and $$p$$, and $$k$$ is a filter constant that satisfies $$k \geq 1$$ and is often taken to be $$1$$.\
Since the photons are assumed to be on two-dimensional surfaces, the cone filter can be normalized with $$(1-\frac{2}{3k})$$. After applying the cone filter, the radiance estimate equation becomes:\
------------------------------------------------------------------------------------------------------------
\$$
L_r(x,\vec{\omega}) \approx \frac{\sum_{p=1}^n f_r(x,\vec{\omega}_p, \vec{\omega}) \Delta\Phi_p(x,\vec{\omega}_p) w\_{pc}}{(1-\frac{2}{3k})\pi r^2}
\$$
This is Equation (7.9) of [[1]](#1) and Equation (5) of [[2]](#2)\
------------------------------------------------------------------------------------------------------------

| ![Ring - Caustics](/images/photon_mapping/Ring_Caustic_2048.png){:width="360" :height="360"} |
|:--:|
| Cardioid Caustic of a Reflective Ring, Rendered using the Cone Filter |

## Participating Media
Before delving into volumetric photon mapping, I will provide a brief overview of the concept of **participating media** in computer graphics. This summary is based on what I learned from Chapter 11 of *Physically Based Rendering: From Theory to Implementation* (PBRT) (4th Edition) [[4]](#4).\
Paticipating media refer to regions filled with small particles that interact with light as it travels through the medium. Fog and cloudy water are common real-world examples of paticipating media. Due to the massive number of particles present in such media, it is infeasible to model each particle individually. Instead, when rendering a scene with participating media, the media's effects on light are modeled probablistically.

### Physical Processes in Participating Media
These are three major physical processes that influence light transport in participating media. I will quote their definitions from PBRT [[4]](#4):
* **Absorption**: the reduction in radiance due to the conversion of light to another form of energy, such as heat.
* **Emission**: radiance that is added to the environment from luminous particles.
* **Scattering**: radiance heading in one direction that is scattered to other directions due to collisions with particles.

In my implementation of participating media, I made the simplification assumptions that all media are non-emissive and **homogeneous** -- meaning the properties of a medium are constant throughout its region.

### Coefficients and Transmittance
* **Absorption, Scattering, and Extinction Coefficients**\
The **absorption coefficient** $$(\sigma_a)$$ of a medium represents the probability per unit distance that a light ray traveling through the medium will be absorbed upon hitting a particle.\
Similarly, the **scattering coefficient** $$(\sigma_s)$$ represents the probability per unit distance that a light ray traveling through the medium will hit a particle and scatter towards a different direction (**out scattering**).\
Both of these interactions reduce the ray's radiance. Thus, the sum of $$\sigma_a$$ and $$\sigma_s$$ is known as the **extinction / attenuation coefficient**, denoted as $$\sigma_t$$.\
Another important quantity is the ratio of the scattering coefficient to the extinction coefficent $$(\frac{\sigma_s}{\sigma_t})$$. This value is called the **single-scattering albedo**, and it will play a key role in the Russian roulette process for volumetric photon mapping, which will be introduced in the next section.

* **Transmittance and Beer's Law**\
For any single light ray, the events of scattering or absorbtion intuitively either occur or not. However, in participating media, these interactions are modeled as continuous reduction in radiance, which statistically achieves the same overall effect.\
The fraction of radiance that remains as a light ray travels from point $$p$$ to $$p'$$ is represented by the **transmittance** $$T_r(p \rightarrow p')$$.\
Under the homogeneous medium assumption, the transmittance from $$p$$ to $$p'$$ can be easily computed following **Beer's Law**:\
------------------------------------------------------------------------------------------------------------
\$$
T_r(p \rightarrow p') = e^{-\sigma_t d}
\$$
This is Equation (11.7) of [[4]](#4), where\
$$d$$ is the distance between $$p$$ and $$p'$$\
$$\tau(p \rightarrow p') = \sigma_t d$$ is also known as the **optical depth**\
------------------------------------------------------------------------------------------------------------

### Phase Function
The **phase function** for participating media is analogous to the BSDF for surfaces -- it describes the distribution of outgoing directions $$(\omega_o)$$ given the incident direction $$(\omega_i)$$ in a scattering event. In my implementation, I assume that all participating media are **isotropic**, meaning the distribution of scattering directions is rotationally symmetric. In this case, the phase function depends only on the angle $$\theta$$ between $$\omega_i$$ and $$\omega_o$$.\
I will discuss the specific phase function I used -- the Henyey-Greenstein Phase Function -- in more detail in the next section.

### Implementing the Medium Framework
Prior to implementing participating media and volumetric photon mapping, I first needed to create a `Medium` class and a framework that allows rays and photons to track their current medium.\
The `Medium` class I developed includes a `ScatteringCoefs` struct to store $$\sigma_s$$, $$\sigma_a$$, $$\sigma_t$$, and the single-scattering albedo $$\frac{\sigma_s}{\sigma_t}$$. The class also includes a `PhaseFunction` class, the medium's refractive index, and several boolean variables to track properties such as whether the medium requires ray marching. Certain media like air or glass do not "participate" in the light transport process beyond their surfaces and thus do not require ray marching. I will describe the ray marching process in the next section.\
To enable rays/photons to track the medium they are currently in, I implemented a medium stack for each ray/photon, initialized with its initial medium. When a ray/photon enters a new medium, the new medium is pushed onto the stack; when it exits the current medium, it is popped off the stack.\
Each primitive in the scene, whether a closed geometry (e.g., a sphere) or an open one (e.g., a triangle), is assigned a `Medium` corresponding to the medium it "surrounds," which is the medium on the side opposite to its normal. When a ray/photon intersects a primitive, if it hits the side aligned with the normal, it is entering a new medium; otherwise, it is exiting the current medium.\
An issue I encountered was the "leaking" of rays/photons at medium boundaries. Due to floating-point precision errors, rays/photons could, in extreme cases, cross a medium boundary unexpectedly -- possibly between the edges of two triangles that are supposed to be perfectly-sealed. This issue is less likely to occur when the medium is enclosed by a single, closed surface (such as a sphere). However, in the box-with-water scene, the water surface is composed of approximately six million triangles.\
To address this problem, I implemented additional checks during medium stack operations. For example, when a ray/photon intersects a primitive and exits a medium, it verifies whether the primitive's medium matches its current medium. If a mismatch occurs, the tracing process for that photon/ray is terminated. Due to the rarity of leaking events, discarding a single photon/ray has no noticeable impact on the overall render.

| ![Water - Medium Framework](/images/photon_mapping/Water_PhotonMapping_2048.png){:width="360" :height="360"} |
|:--:|
| The Box with Water Scene after Implementing the Medium Framework (Before Volumetric Photon Mapping) |
|:--:|
| The scene features a glass dome submerged underwater,<br>with a reflective sphere positioned half inside and half outside the glass dome. |

## Volumetric Photon Mapping
The photon mapping algorithm was extended to handle participating media by Dr. Henrik Wann Jensen and Dr. Per H. Christensen in their 1998 paper, "Efficient Simulation of Light Transport in Scenes with Participating Media using Photon Maps" [[3]](#3). While the paper did not coin the term **volumetric photon mapping**, I have adopted it from the 2008 paper, "The Beam Radiance Estimate for Volumetric Photon Mapping," by Dr. Wojciech Jarosz, Dr. Matthias Zwicker, and Dr. Henrik Wann Jensen [[5]](#5).

### 1st Pass: Photon Tracing

* **Photon Scattering**\
TODO

* **Photon Storing**\
TODO\
**volume photon map**

### 2nd Pass: Rendering

* **Volume Radiance Estimate**\
TODO\
**in scattering**

* **Ray Marching**\
TODO

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
  <div>
    <a id="4">[4]</a>
    Matt Pharr, Wenzel Jakob, and Greg Humphreys. 2023. Physically Based Rendering: From Theory to Implementation (4th ed.). MIT Press, Cambridge, MA, USA.
  </div>
  <div>
    <a id="5">[5]</a>
    Wojciech Jarosz, Matthias Zwicker, and Henrik Wann Jensen. 2008. The beam radiance estimate for volumetric photon mapping. In ACM SIGGRAPH 2008 classes (SIGGRAPH '08). Association for Computing Machinery, New York, NY, USA, Article 3, 1–112. https://doi.org/10.1145/1401132.1401137
  </div>
</div>
