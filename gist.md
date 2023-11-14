# GPU-driven rendering

The most important term you need to know is 'gpu-driven rendering'.

With a regular 'cpu-driven' pipeline the cpu has the authority on whats being rendered. It culls objects and issues commands like bind this texture, bind that texture, draw this, draw that etc. In this situation the GPU only recieves information about what the scene is like in bits and pieces, and can spend a significant amount of time switching between different textures and buffers, both of which add overhead. In technical terms, both the CPU and GPU are doing O(n) work with respect to the amount of geometry and textures.

In a gpu-driven pipeline, the GPU has all the scene information available to it simultaneously, and is able to manage it's own work via writing to an on-gpu buffer of draw calls to make. Objects can be culled beforehand via compute shaders without any information needing to be read back to the CPU. The CPU does O(1) work, basically just issuing a few compute shader dispatches and indirect draw calls, while the GPU can get on and render everything without overhead.

Here's a comparison between what a CPU-driven pipeline could look like (in pseudocode):

```python
for model in models {
    bind_appropriate_render_pipeline(cmdbuffer);
    bind_descriptor_set(model, cmdbuffer);
    bind_vertex_and_index_buffers(model, cmdbuffer);
    num_instances, instance_buffer = instance_buffer_for_model(model);
    bind_instance_buffer(instance_buffer, cmdbuffer);
    draw(num_instances, model.num_vertices);
}
```

and a GPU-driven pipeline:

```python
bind_all_buffers(cmdbuffer); # but implemented how?
bind_combined_model_descriptor_set(cmdbuffer); # but implemented how?

# Render pipelines are roughly merged together but still differentiated when it makes sense.
# Here we're dividing up geometry by blend mode, so all the opaque objects are drawn first,
# then all the alpha-clipped objects, etc.

bind_render_pipeline(cmdbuffer, opaque_pipeline); 
draw_all_opaque_models(); # but implemented how?

bind_render_pipeline(cmdbuffer, alpha_clip_pipeline); 
draw_all_alpha_clip_models();

bind_render_pipeline(cmdbuffer, alpha_blend_pipeline); 
draw_all_alpha_blend_models();
```

There are 3 main requirements to unlocking GPU-driven rendering:
1. So-called 'bindless' texturing and buffers. I take this term to mean that you literally want to 'bind (textures/buffers) less' as opposed to never at all.
2. Merging pipelines together, by handing multiple materials or shading techniques with the same pipeline where possible. There are still reasons to differentiate pipelines, blending mode (opaque/alpha clipped/alpha blended) and animation state (static/skinned) being two examples. For a benchmark, [Doom Eternal used around 500 pipelines](https://vkguide.dev/docs/gpudriven/gpu_driven_engines/).
3. With the above two out of the way, you can yoink the pipeline, buffer and descriptor set bindings out of the `for` loop and you're just left with the draw calls, making them perfect targets for the second best API call in Vulkan, [`vkCmdDrawIndexedIndirect`](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/vkCmdDrawIndexedIndirect.html), only followed by it's even better sibling, [`vkCmdDrawIndexedIndirectCount`](https://registry.khronos.org/VulkanSC/specs/1.0-extensions/man/html/vkCmdDrawIndexedIndirectCount.html). More on those later!

# Bindless

## Bindless geometry

There are roughly two ways to handle bindless geometry:

### Concatonate all geometry into big index + vertex buffers
- This works fine until you want to deallocate any geometry. If you store the ranges of where inside the buffer of where geometry is stored (e.g. `85337..190000`) then you can add that range to a list of empty gaps inside the buffer, and check that list when doing allocations. I've done this in the past.
- Handling defragmentation sounds painful though.

### Use a seperate buffer for each piece of geometry and use `VK_KHR_buffer_device_address` to access the buffers in shaders.
  - This is way nicer than the above IMO because it means you can rely on the allocator of your choice (e.g. VMA) to handle allocations and deallocations.
  - The main downside is that you are relying on `VK_KHR_buffer_device_address` (available on pretty much any devices) and that you have to manually access the index/vertex buffers in the vertex shader.

## Bindless textures

There's `VK_NVX_image_view_handle` which essentially does the same sorta thing as `VK_KHR_buffer_device_address` but it's Nvidia-only (for now) so we can ignore it.

### Megatextures

Yeah, no, these kinda suck unfortunately. They seem hard to manage and had a bunch of pop-in and other issues. RAGE had them, Doom 2016 had them (but less afaik?) and Doom Eternal got rid of them: https://www.doomworld.com/forum/topic/112067-so-megatextures-are-no-more-in-id-tech-7/

### Descriptor indexing / Arrays of textures

It's the better option! Still not great, but better.

The idea here is that you have a large array of image descriptors in a descriptor set that you index into dynamically in a shader. The terminology sucks here, as hopefully indicated by the below image. Handling new images is pretty easy, just add the image to the descriptor set and get it's index for accessing. For removals, you want to record that the index is empty and can be re-used. This works on most modern hardware. I think Macs had some problem with it a whileback (via MoltenVK) but that seems to no longer be an issue.

![onion_textures_know_the_diff](https://user-images.githubusercontent.com/13566135/282337180-11e53f09-ea70-49a6-bd0f-3dacb264393d.png)
_from https://chunkstories.xyz/blog/a-note-on-descriptor-indexing/_

# Culling

- Compute shader takes in scene geometry and their bounding sphere and culls it if it's not visible in the view frustum.
- Could also cull against the previous frame's hierarchical depth buffer (see that Assassin's Creed: Unity slidedeck).
- Could also do LoD selection.
- Writes to a storage buffer with atomics.
- `vkCmdDrawIndexedIndirectCount` is called.

## Mesh shaders

Mesh shaders let you stream geometry from the scene description to the rasterizer, doing culling and LoD selection along the way. Afaik this is more flexible and faster as it doesn't involve a buffer write and readback.

# Forwards or deferred?

A: They both suck in their naïve implementations.

If you have 

`P` = the number of pixels being rendered to  
`L` = the number of lights in the scene  
`G` = the number of geometry being written  
`log2(G)` = roughly the amount of overdraw per pixel  

then forwards is `O(P . L . log2(G))` and deferred is `O(P . log2(G) + P . L)`. Deferred is clearly better here. But it is super heavy in terms of the number of render targets it requires, especially for PBR where you have albedo+normal+metallic+roughness+(any other materials params, emission, IoR, clearcoat).

A few more modern options:

## Forwards with a depth pre-pass

- Still `O(P . log2(G) + P . L)`, but doesn't require any more render targets
- The depth pre-pass is super easy to intergrate, especially when doing bindless rendering
- Does have the downside of requiring the vertex shader and rasterizer be run twice for everything though

## Deferred texturing

- Instead of storing texture data in the deferred pass, store the data needed to read it!
- Could be a various combination of things but generally involves the triangle barycentrics (a `vec2`). Could be the Material ID + normal + barycentrics OR mesh id + triangle id + barycentrics or anything else
- `VK_KHR_fragment_shader_barycentric` makes implementation easier but I'm not sure how supported it is.
- See that 'deferred texturing in horizon forbidden west slidedeck'. 
