---
title: Light Rendering Techniques
date: 2026-01-15 12:00:00 +0100
categories: [ articles ]
tags: [ graphics-programming, cpp, deferred, tiled, forward+ ]
description: Diving into different techniques to render a large number of lights efficiently
image:
  path: /assets/media/finalresult.png
---

## Introduction

Light rendering has historically been known to heavily impact game performance. Ever since then, people have come up with improvements to pre-existing techniques as well as new techniques altogether for rendering a large number of dynamic lights. In this article I want to present and delve into these alternative techniques, as well as talk a bit about why someone might choose one over another. 

**Disclaimer:** The rendering API used in this article is DirectX 12, using Microsoft's MiniEngine. I will be as API-agnostic as I can be when talking about these techniques, but code snippets of the pipeline setup will still show API calls. The shading model used will be a PBR model based on Filament.

## Forward rendering

The most common technique as well as the simplest, traditional forward rendering has been used as a default approach for most 3D renderers for a couple of decades now. 

### Breakdown

The way this technique's pipeline is structured is very simple: as soon as a mesh is drawn, it's also shaded. It's a very straighforward approach, usable for many scenarios. There's a reason it's been the default technique used for so long.

### Implementation

As mentioned before, this is pretty straightforward: in the context, we set the root signature and pipeline state, clear the color and depth buffer, do a few more necessary API calls and then draw the objects.

```cpp
gfxContext.SetRootSignature(forwardRootSignature);
gfxContext.SetPipelineState(forwardShadingPSO);
gfxContext.TransitionResource(g_SceneColorBuffer, D3D12_RESOURCE_STATE_RENDER_TARGET, true);
gfxContext.TransitionResource(g_SceneDepthBuffer, D3D12_RESOURCE_STATE_DEPTH_WRITE, true);
gfxContext.ClearColor(g_SceneColorBuffer);
gfxContext.ClearDepth(g_SceneDepthBuffer);
gfxContext.SetRenderTarget(g_SceneColorBuffer.GetRTV(), g_SceneDepthBuffer.GetDSV());
gfxContext.SetViewportAndScissor(0, 0, g_SceneColorBuffer.GetWidth(), g_SceneColorBuffer.GetHeight());
gfxContext.SetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
gfxContext.SetDynamicDescriptor(3, 0, lightBuffer.GetSRV());

DrawObjects(gfxContext);
```

As for the shaders, there's only two: a vertex and a pixel shader. These are nothing special. The vertex shader takes in the mesh data, does a few calculations using it, and then sends data to the pixel shader that it can use to shade the mesh.

## Deferred rendering

This technique was introduced as a way to overcome Forward rendering's drawbacks, especially when it comes to more complex scenes. The deferred pipeline is vastly different and more complicated than traditional Forward rendering, which can be a bit daunting at first, but is actually relatively simple.

### Breakdown

The pipeline consists of two passes: **the geometry pass** and **the lighting pass**. Essentially, the idea behind deferred rendering comes from the name: we defer the actual shading of the pixels to a later stage. 

**The geometry pass** takes in all the mesh data and, in its pixel shader, writes it into something called the **G-Buffer**, which is short for **Geometry Buffer**. Simply put, the G-Buffer is just a collection of textures, each containing different data required for the lighting calculations of the mesh. The geometry pass goes over all of the meshes in the scene, drawing into the G-Buffer textures. At the end of the geometry pass, we will have our textures full of the final geometry data of our scene. 

The image below shows an example of the geometry pass output, specifically the albedo textures. We are now ready for the next step.

![Geometry pass output](/assets/media/geometrypassoutput.jpg)

**The lighting pass** reads from the textures we drew into in the geometry pass and uses them to perform lighting calculations, which are done the same way you'd do them in Forward rendering. As such, the image above is not only the output of the geometry pass, but also the input of the lighting pass. At the end of the lighting pass, we have our final image.

![Lighting pass output](/assets/media/lightingpassoutput.png)

### Implementation

In my implementation, the structure and layout of my G-Buffer was chosen for its simplicity and ease of use, rather than out of a necessity to squeeze out performance. The textures that encompass my G-Buffer are: the geometry normals, albedo texture, emissive texture, metallic roughness texture, and the world position of the pixels.

First, let's set up the **geometry pass**:

```cpp
gfxContext.SetRootSignature(geometryPassRootSignature);
gfxContext.SetPipelineState(geometryPassPSO);
gfxContext.TransitionResource(geometryPassNormalBuffer, D3D12_RESOURCE_STATE_RENDER_TARGET, true);
gfxContext.TransitionResource(geometryPassAlbedoBuffer, D3D12_RESOURCE_STATE_RENDER_TARGET, true);
gfxContext.TransitionResource(geometryPassWorldPositionBuffer, D3D12_RESOURCE_STATE_RENDER_TARGET, true);
gfxContext.TransitionResource(geometryPassEmissiveBuffer, D3D12_RESOURCE_STATE_RENDER_TARGET, true);
gfxContext.TransitionResource(geometryPassMetallicRoughnessBuffer, D3D12_RESOURCE_STATE_RENDER_TARGET, true);
gfxContext.TransitionResource(g_SceneDepthBuffer, D3D12_RESOURCE_STATE_DEPTH_WRITE, true);
gfxContext.ClearColor(geometryPassNormalBuffer);
gfxContext.ClearColor(geometryPassAlbedoBuffer);
gfxContext.ClearColor(geometryPassWorldPositionBuffer);
gfxContext.ClearColor(geometryPassEmissiveBuffer);
gfxContext.ClearColor(geometryPassMetallicRoughnessBuffer);
gfxContext.ClearDepth(g_SceneDepthBuffer);
gfxContext.SetRenderTargets(_countof(gBufferRenderTargets), gBufferRenderTargets, g_SceneDepthBuffer.GetDSV());
gfxContext.SetViewportAndScissor(0, 0, g_SceneColorBuffer.GetWidth(), g_SceneColorBuffer.GetHeight());
gfxContext.SetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);

DrawObjects(gfxContext);
```

While this may seem like an overwhelming number of API calls, they do practically the same thing as the ones in the implementation of forward rendering, just on a larger scale. Instead of clearing one color buffer, we clear as many as we have G-Buffer textures. Then, we just draw the objects. 

This is an example of how you'd write into one of the G-Buffer textures inside of the pixel shader:
```hlsl
struct GeometryBuffer
{
    float4 normals : SV_Target0;
    float3 albedo : SV_Target1;
    float3 worldPosition : SV_Target2;
    float3 emissive : SV_Target3;
    float2 metallicRoughness : SV_Target4;
};


GeometryBuffer main(PixelShaderInput input)
{
  GeometryBuffer gBuffer;
    
    float3 baseTexture = baseTexture.Sample(mainSampler, input.uv);
    
    gBuffer.albedo = baseTexture.xyz * baseColorFactor;
  
    ...

    return gBuffer;
}
```

By the end of this pass, our textures are already full with all the information we need to move onto the next step.

The **lighting pass** is very simple:

```cpp
gfxContext.TransitionResource(geometryPassAlbedoBuffer, D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE);
gfxContext.TransitionResource(geometryPassNormalBuffer, D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE);
gfxContext.TransitionResource(geometryPassWorldPositionBuffer, D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE);
gfxContext.TransitionResource(geometryPassEmissiveBuffer, D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE);
gfxContext.TransitionResource(geometryPassMetallicRoughnessBuffer, D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE);
gfxContext.TransitionResource(g_SceneColorBuffer, D3D12_RESOURCE_STATE_RENDER_TARGET);
gfxContext.ClearColor(g_SceneColorBuffer);
gfxContext.SetRootSignature(lightingPassRootSignature);
gfxContext.SetPipelineState(lightingPassPSO);
gfxContext.SetDynamicDescriptor(0, 0, geometryPassNormalBuffer.GetSRV());
gfxContext.SetDynamicDescriptor(1, 0, geometryPassAlbedoBuffer.GetSRV());
gfxContext.SetDynamicDescriptor(2, 0, geometryPassWorldPositionBuffer.GetSRV());
gfxContext.SetDynamicDescriptor(3, 0, geometryPassEmissiveBuffer.GetSRV());
gfxContext.SetDynamicDescriptor(4, 0, geometryPassMetallicRoughnessBuffer.GetSRV());
gfxContext.SetDynamicDescriptor(5, 0, lightBuffer.GetSRV());
gfxContext.SetConstantArray(6, sizeof(glm::vec3) / sizeof(uint32_t), &camera->position);
gfxContext.SetRenderTarget(g_SceneColorBuffer.GetRTV());
gfxContext.SetVertexBuffer(0, quadVertexBufferView);

gfxContext.Draw(6);
```

We transition the former render targets into pixel shader resources, clear the main color buffer and bind the G-Buffer textures to the shader, as well as any other data we may need. Then, we draw a screen-filling quad.

This is an example of how you'd use the data passed from the G-Buffer in the pixel shader:

```cpp
...
Texture2D<float3> geometryAlbedo : register(t1);
...

float3 main(PixelShaderInput input) : SV_Target
{
    ...
    
    float3 albedo = geometryAlbedo.Sample(mainSampler, input.uv);
    
    ...

    // Lighting calculations

    ...
}
```

That pretty much covers it. As soon as this part of the pipeline is done, our work is done and the final image is presented.

## Forward+

### Breakdown

### Implementation

## Comparison

## Conclusion


## Resources

[^1]: [A Primer On Efficient Rendering Algorithms & Clustered Shading](https://www.aortiz.me/2018/12/21/CG.html)
[^2]: [Forward vs Deferred vs Forward+ Rendering with DirectX 11](https://www.3dgep.com/forward-plus/)
