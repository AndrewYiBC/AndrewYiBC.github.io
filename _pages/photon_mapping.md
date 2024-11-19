---
permalink: /photon_mapping/
title: "Volumetric Photon Mapping Project"
author_profile: true
redirect_from: 
  - /photon/
  - /photon_mapping/
  - /photon_mapping.html
---

In computer graphics, **caustics** refer to patterns of light created by light rays focusing onto specifc areas of a surface, often through specular reflections or refractions. When such light rays transmit through a participating medium, they can be scattered and create **volume caustics**. In real life, caustics can be observed under a wavy water surface or in the shadow of a curved glass of liquid. An example of volume caustics could be the sun casting beams of light through a misty morning forest -- a scene I witnessed during a visit to Sequoia National Park in November 2023, which also served as the inspiration of this project.

Drawn to the astonishing visual effects of caustics and volume caustics, I decided to implement **photon mapping** and **volumetric photon mapping**, and I was fortunate to have Professor [Theodore Kim](https://seas.yale.edu/faculty-research/faculty-directory/theodore-kim) supervise my project. I implemented this global illumination algorithm based on a series of publications by Dr. Henrik Wann Jensen and Dr. Per H. Christensen [[1]](#1)[[2]](#2)[[3]](#3). In the sections following the demo gallery, I will provide a brief overview of the photon mapping and volumetric photon mapping algorithms, along with some details of my implementation of the algorithms and the "box with water" demo scene, as well as the challenges I faced during the process.

## Gallery

| ![Water - Volumetric Photon Mapping](/images/photon_mapping/Water_VolumetricPhotonMapping_2048.png){:width="360" :height="360"} | ![Water - Photon Mapping](/images/photon_mapping/Water_PhotonMapping_2048.png){:width="360" :height="360"} |
|:--:|:--:|
| Box with Water Scene 1 - Volumetric Photon Mapping | Box with Water Scene 2 - Photon Mapping |

| ![Fog - Volume Caustics](/images/photon_mapping/Fog_VolumeCaustic_1024.png){:width="360" :height="360"} | ![Ring - Caustics](/images/photon_mapping/Ring_Caustic_2048.png){:width="360" :height="360"} |
|:--:|:--:|
| Volume Caustic through a Fog Medium | Cardioid Caustic of a Reflective Ring |

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
