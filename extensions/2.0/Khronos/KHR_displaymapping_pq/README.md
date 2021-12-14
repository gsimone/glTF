# KHR_displaymapping_pq

## Contributors


Rickard Sahlin, <mailto:rickard.sahlin@ikea.com>  
Sebastien Vandenberghe, <mailto:sevan@microsoft.com>  

Copyright (C) 2021 The Khronos Group Inc. All Rights Reserved. glTF is a trademark of The Khronos Group Inc.
See [Appendix](#appendix-full-khronos-copyright-statement) for full Khronos Copyright Statement.

## Status

Draft

## Dependencies

Written against the glTF 2.0 spec.

## Exclusions


## Overview

The goal of this extension is to provide the means to map internal, usually floating point, light contribution values that may be in an unknown range to that of a known range and attached display.  
One of the reasons for this is to retain the hue of the source materials under varying light conditions.  
Correct representation of hue is important in order to keep artistic intent, or to achieve a physically correct visualization of products for instance in e-commerce.  

It also provides the specification for using HDR compatible display outputs while at the same time retaining compatibility with SDR display outputs.  
The intended usecases for this extension is any usecase where the light contribution values will go above 1.0, for instance by using KHR_lights_punctual, KHR_emissive_strength or KHR_environment_lights.  

This extension has three integration points:  
1: If the framebuffer colorspace is compatible with ITU BT.2020 color primaries, then before images are sampled they shall be converted to ITU BT.2020 colorspace  
This can be done from linear colorspace by the conversion matrix outlined later in this document - [see 'To HDR capable display'](#to-hdr-capable-display)  

2: The scene max light intensity, ie the highest light value, needs to be approximated when automatic aperture is enabled.  
The goal of this is to provide a means to increase or decrease the overall brightness of the scene.  
This does not have to be done in an exact manner every frame, approximations or slight delays in adjustment are allowed.  
[See `sceneAperture` parameter](#parameters)  

3: Before outputing the calculated pixel value, ie the light reaching the viewer from a point in the scene, the opto-optical transfer function shall be applied followed by the opto-electrical transfer function.  
This is the process that will adjust for gamma and adapt the 0 - 10000 range value into a nonlinear display value.  

[See OOTF](#ootf)  
[See OETF](#oetf)  


### Motivation

Output pixel values from a rendered 3D model are generally in a range that is larger than that of a display device.  
This may not be a problem if the output is a high definition image format or some other target that has the same range and precision as the internal calculations.  
However, a typical usecase for realtime rasterizer implementations is that the output is a light emitting display.  
Such a display rarely has the range and precision of internal calculations making it necessary to map internal pixel values to match the characteristics of the output.  
This mapping is generally referred to as tone-mapping, however the exact meaning of tone-mapping varies and can also mean the process of applying artistic intent to the output.  
For that reason this document will use the term displaymapping.    

The displaymapping for this extension is chosen from ITU BT.2100 which is the standard for HDR TV and broadcast content creation.   
This standard uses the perceptual quantizer as transfer function, ie to go from scene linear values to non linear output values.  
The function is selected based on minimizing visual artefacts from color banding according to the Barten Ramp. Resulting on very slight visible banding on panels with 10 bits per colorchannel.  
On panels with 12 bits there is no visible banding artefacts when using the perceptual qantizer.  

Apart from being widely supported and used in the TV / movie industry it is also embraced by the gaming community, with support in the engines from some of the major game companies.  
For instance, game engines Frostbite and Lumberyard and also in specific games such as Destiny 2 and Call Of Duty.  


Here the term rasterizer means a rendering engine that consits of a system wherein a buffer containing the pixel values for each frame is prepared. 
This buffer will be referred to as the framebuffer.  
The framebuffer can be of varying range, precision and colorspace. This has an impact on the color gamut that can be displayed.  
  
After completion of one framebuffer it is output to the display, this is usually done by means of a swap-chain. The details of how the swap works is outside the scope of this extension.  
KHR_displaymapping_pq specifies one method of mapping internal pixel values to that of the framebuffer.  

This extension does not take the viewing environment of the display, or eye light adaptation, into consideration.  
It is assumed that the content is viewed in an environment that is dimly lit (~5 cd / m2) without direct light on the display.  


## Internal range

When the KHR_displaymapping_pq extension is used all lighting and pixel calculations shall be done using the value 10000 (cd / m2) as the maximum ouput brightness.  
This does not have an impact on color texture sources since they define values as contribution factor.  
The value 10000 cd / m2 for an output pixel with full brightness is chosen to be compatible with the Perceptual Quantizer (PQ) used in the SMPTE ST 2084 transfer function.  

When using this extension light contribution values shall be aligned to account for 10000 cd/m2 as max luminance.  
This means that content creators shall be aware of 10000 cd/m2 as the maximum brightness value range, it does not mean that the display will be capable of outputing at this light luminance.  

## Scene light value

A content creation tool supporting this extension shall sum upp light contribution for a scene before exporting to glTF, this can be a naive addition of all lights included in the scene that adds max values together.  
If scene max light contribution is above 10000 cd / m2 there is a choice to either downscale light values before export or to set `sceneAperture` to `AUTO`  
This means that implementations will calculate max light intensity for the scene and use as a scale factor to keep all light contribution within 0 - 10000 cd / m2.  

Light contribution values above 10000 cd / m2 is strongly discouraged and will be clamped by implementations if not adjusted by scene aperture.    

The internal precision shall at a minimum match the equivalent of 12 bits unsigned integer, which maps to the PQ.  

## Displaymapping

### To HDR capable display

To convert from internal values (linear scene light) to the non-linear output value in the range 0.0 - 1.0 the PQ EOTF shall be used.  
This is specified in ITU BT.2100:
https://www.itu.int/rec/R-REC-BT.2100/en  

If the framebuffer format and colorspace is known to the implementation then a format and colorspace shall be chosen to preserve the range and precision of the SMPTE ST 2084 transfer function.  
If available, a framebuffer colorspace that is compatible with the color primaries of ITU BT.2020 shall be used.  
If this colorspace is used the source color images must be converted from linear ITU BT.709 values to ITU BT.2020 linear values, where applicable depending on texture target and source format.  

**Implementation notes**

Conversion of color primaries to ITU BT.2020 could be done after loading of a PNG/JPEG and after the image has been gamma expanded from sRGB to linear.  

#### Color conversion matrix

Color conversion from BT.709 to BT.2020 is specified in ITU BT.2087  
https://www.itu.int/rec/R-REC-BT.2087/en

The M2 linear color conversion matrix is defined as:  
0.6274 0.3293 0.0433  
0.0691 0.9195 0.0114  
0.0164 0.0880 0.8956  

A HDR capable display is defined as having at least 10 bits per pixel for each colorchannel. 

### To SDR capable display

If the framebuffer format or colorspace is not known, or none is available that preserves range, precision and color gamut then lower range framebuffer with sRGB colorspace may be used.  
This is to allow for compatibility with displays that does not support higher range and/or compatible colorspaces.  
It also allows for implementations where the details of the framebuffer is not known or available.  

A SDR capable display is defined as having less than 10 bits per pixel for each colorchannel.  


### OOTF

Resulting linear scene light values shall be display mapped according to the parameter `Reference PQ OOTF` of ITU BT.2100   
The opto-optical reference transform shall be applied, the reference transform compatible with both SDR and HDR displays is described in ITU BT.2390 with the use of range extension and gamma values.  

E′ = G709[E] = pow(1.099 (rangeExtension * E), 0.45) – 0.099 for 1 > E > 0.0003024
               267.84 * E for 0.0003024 ≥ E ≥ 0
FD = G1886[E'] = pow(100 E′, gamma)

E is the linear scene light value. 

Where the rangeExtension and gamma values can be set by this extension for HDR and SDR display outputs.  
* For HDR output,  a range extension value of 59.5208 and gamma of 2.4 is suggested, these values can be changed according to artistic intent.  
Some game engines are known to use a value of 2.8 for HDR gamma.  

* For SDR output,  a range extension value of 46.42 and gamma of 2.4 is suggested, these values can be changed according to artistic intent.  

**Implementation Notes**

Pseudocode for BT.2100 reference OOTF  

BT_2100_OOTF(color, rangeExponent, gamma) {  
    if (color <= 0.0003024) {  
        nonlinear = 267.84 * color;  
    else {  
        nonlinear = 1.099 * pow(rangeExponent * color, 0.45) - 0.099;  
    }  
    return 100 * pow(nonlinear, gamma);
}  

Where this is calculated per RGB component, note that the color value in this equation must be in range 0.0 - 1.0
It may be estimated that the display will not have the ability to output dark levels in the region of 0.0003024 in which case implementations may ignore that condition and use the same operator for the whole range 0.0 - 1.0  


### OETF

After the OOTF is applied the OETF shall be applied, this will yield a non linear output-signal in the range [0.0 - 1.0] that is stored in the display buffer.  
This shall be done according to the parameter `Reference PQ OETF`of ITU BT.2100    

Where the resulting non-linear signal (R,G,B) in the range [0:1] = ((C1 + C2 * pow(FD / 10000, m1)) / (1 + C3 * pow(FD / 10000, m1))  ^ m2 

FD from the OOTF and  
m1 = 2610/16384 = 0.1593017578125 
m2 = 2523/4096 * 128 = 78.84375 
c1 = 3424/4096 =0.8359375 = c3 − c2 + 1
c2 = 2413/4096 * 32 = 18.8515625
c3 = 2392/4096 * 32 = 18.6875



### Parameters

The following parameters are added by the `KHR_displaymapping_pq` extension:

| Name                   | Type       | Description                                    | Required           |
|------------------------|------------|------------------------------------------------|--------------------|
| **ootfHDR**         | `object`   | Opto-optical transfer function values for HDR display output, if no value is specified then the default parameters rangeExtension=59.5208 and gamma=2.4 is used  | No |
| **ootfSDR**         | `object`   | Opto-optical transfer function values for SDR display output, if no value is specified then the default parameters rangeExtension=46.42 and gamma=2.4 is used  | No |
| **sceneAperture**   | `object`   | Scene light adjustment setting, if no value supplied the default value of OFF will be used and scene light values will be kept unmodified.    | No |


`sceneAperture`

| **apertureValue**   | `number`   | Aperture factor used to calculate aperture factor  | No |
| **apertureControl**   | `number`   | Aperture control value used to calculate aperture factor  | No |
| **minLuminance**   | `number`   | Min luminance to use when calculating aperture factor, if no value is specified it defaults to 0  | No |
| **maxLuminance**   | `number`   | Max luminance to use when calculating aperture factor, if no value is specified it defaults to 10000  | No |


This can be seen as a simplified automatic exposure control, without shutterspeed and ISO values, it can be seen as a way to get similar result to how the human eye would adapt to different light conditions by letting in more or less light.  
The intended usecase is simpler scenes that do not change drastically, for instance when displaying product models or a room.  
It is not intended for gaming like usecases where the viewpoint and environment changes dramitically, for instance from dimly lit outdoor night environments to a highly illuminated hallway.  

This setting is a way to avoid having too high scene brightess, that would result in clamping of output values.  
Or to avoid a scene from being too dark.  

luminance = clamp(luminance, minLuminance, maxLuminance) 
Aperture = apertureValue / (luminance - luminance * apertureControl)

Before values are displaymapped, ie before the OOTF and OETF is applied, pixel values shall be factored by the Aperture value.  


**Implementation Notes**

Since knowledge of overall scene brightness values may be time-consuming to calculate exactly, implementations are free to approximate.  
This could be done by calculating the max scene brightness once, adding up punctual and environment lights and then using this value, not taking occlusion of lightrays into account.    


## Schema

[glTF.KHR_displaymapping_pq.json](schema/glTF.KHR_displaymapping_pq.json)

The `KHR_displaymapping` extension is added to the root of the glTF   

```json
{
  "extensions": {
    "KHR_displaymapping_pq" : {
      "ootfHDR": {
        "rangeExtension": "58",
        "gamma": "2.8"
      },
      "ootfSDR": {
        "rangeExtension": "46",
        "gamma": "2.2"
      },
      "sceneAperture": {
        "apertureValue": "5000",
        "apertureControl": "0",
        "minLuminance": "300",
        "maxLuminance": "5000"
      }
    }
  },
  "extensionsUsed": [
    "KHR_displaymapping_pq"
  ]
  }
```


## Appendix: Full Khronos Copyright Statement

Copyright 2021 The Khronos Group Inc.

Some parts of this Specification are purely informative and do not define requirements
necessary for compliance and so are outside the Scope of this Specification. These
parts of the Specification are marked as being non-normative, or identified as
**Implementation Notes**.

Where this Specification includes normative references to external documents, only the
specifically identified sections and functionality of those external documents are in
Scope. Requirements defined by external documents not created by Khronos may contain
contributions from non-members of Khronos not covered by the Khronos Intellectual
Property Rights Policy.

This specification is protected by copyright laws and contains material proprietary
to Khronos. Except as described by these terms, it or any components
may not be reproduced, republished, distributed, transmitted, displayed, broadcast
or otherwise exploited in any manner without the express prior written permission
of Khronos.

This specification has been created under the Khronos Intellectual Property Rights
Policy, which is Attachment A of the Khronos Group Membership Agreement available at
www.khronos.org/files/member_agreement.pdf. Khronos grants a conditional
copyright license to use and reproduce the unmodified specification for any purpose,
without fee or royalty, EXCEPT no licenses to any patent, trademark or other
intellectual property rights are granted under these terms. Parties desiring to
implement the specification and make use of Khronos trademarks in relation to that
implementation, and receive reciprocal patent license protection under the Khronos
IP Policy must become Adopters and confirm the implementation as conformant under
the process defined by Khronos for this specification;
see https://www.khronos.org/adopters.

Khronos makes no, and expressly disclaims any, representations or warranties,
express or implied, regarding this specification, including, without limitation:
merchantability, fitness for a particular purpose, non-infringement of any
intellectual property, correctness, accuracy, completeness, timeliness, and
reliability. Under no circumstances will Khronos, or any of its Promoters,
Contributors or Members, or their respective partners, officers, directors,
employees, agents or representatives be liable for any damages, whether direct,
indirect, special or consequential damages for lost revenues, lost profits, or
otherwise, arising from or in connection with these materials.

Khronos® and Vulkan® are registered trademarks, and ANARI™, WebGL™, glTF™, NNEF™, OpenVX™,
SPIR™, SPIR-V™, SYCL™, OpenVG™ and 3D Commerce™ are trademarks of The Khronos Group Inc.
OpenXR™ is a trademark owned by The Khronos Group Inc. and is registered as a trademark in
China, the European Union, Japan and the United Kingdom. OpenCL™ is a trademark of Apple Inc.
and OpenGL® is a registered trademark and the OpenGL ES™ and OpenGL SC™ logos are trademarks
of Hewlett Packard Enterprise used under license by Khronos. ASTC is a trademark of
ARM Holdings PLC. All other product names, trademarks, and/or company names are used solely
for identification and belong to their respective owners.