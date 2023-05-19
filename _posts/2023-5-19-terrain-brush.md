---
layout: post
title: "Implementing a realtime terrain brush"
categories: [Computer Graphics]
tags: [terrain, graphics]
math: true
author: NotAPenguin
image: 
  path: /assets/img/andromeda/brush/demo.png
  alt: Simple brush tool in my terrain editor.
---

In most digital content creation software, brush-like tools are a core way to paint visuals.
This week I decided to take a crack at implementing a first terrain brush in my terrain editor,
[andromeda](https://github.com/NotAPenguin0/andromeda).
At first this seemed like a huge undertaking, so I made a list of individual tasks to complete.

- Find out where in the world the mouse clicked.
- Update the heightmap around this point.
- Render a decal around the mouse to show the active brush.

Individually, these are a bit more manageable.
With my mind at ease, it was time to start on the first item on the list.

## Reconstructing world position

The first task is to find the position of the mouse in world coordinates.
One way to do this is with a raycast, 
but since this would involve duplicating all height data on the CPU I decided not to do this.
Instead, I decided to reconstruct the world position from the depth buffer instead.
This is not very difficult to do luckily, so I added a simple compute shader to my render pipeline.

```hlsl
[numthreads(1, 1, 1)]
void main(uint3 GlobalInvocationID : SV_DispatchThreadID) {
    // Get screenspace uv of the mouse position and sample the depth value
    float2 uv = mouse_screen_pos / float2(rt_width, rt_height);
    float depth = depth_rt.SampleLevel(smp, uv, 0);
    // Position of the mouse in NDC.
    float4 ndc_pos = float4(uv.x * 2.0 - 1.0, uv.y * 2.0 - 1.0, depth, 1.0);
    float4 unprojected_pos = mul(inv_projection, ndc_pos);
    // Apply perspective division
    float4 viewspace_pos = float4(unprojected_pos.xyz / unprojected_pos.w, 1.0);
    float4 worldspace_pos = mul(inv_view, viewspace_pos);
    // Write worldspace_pos to output buffer
}
```
{: file='pos_from_depth.cs.hlsl'}

For the sake of simplicity, I will often omit the shader inputs and outputs from example code.
The way this shader works is by simply doing the inverse of what the vertex shader does when rasterizing the terrain.
One by one we undo the transformation steps, and finally we obtain the worldspace position of the mouse.

```
> Mouse position: [-126.0403, 17.618492, -28.716944, 1]
> Mouse position: [-125.389084, 17.72097, -28.58245, 1]
...
```
{: file='Logged output'}

That seems to be working nicely.
The drawback of this is that is introduces some frame lag, since we can only read back the data from the GPU
once the frame has completed rendering.
As long as the mouse is not moving too fast and the framerate is high enough, you generally won't notice this though.

## Updating the heightmap

Now that we know the mouse position at all times, we are ready to move on to step two: Actually updating the terrain.
In my project, the terrain is rendered by tessellating a completely flat plane and applying the heightmap in the 
tessellation evaluation shader[^fn-tess-eval]. 
This means that to update the terrain the mesh itself can stay the same, we just need to update the height map.

The first step is to map the worldspace position of the mouse click to a point on the heightmap.
We can take advantage of the fact that we are using a square, flat plane as the terrain mesh to calculate UV coordinates
by taking the ratio of the mouse position to the terrain size:

```rust
pub fn uv_at(world_pos: Vec3) -> Vec2 {
    // Get the length of the terrain in each dimension
    let dx = (max_x - min_x).abs();
    let dy = (max_y - min_y).abs();
    // The terrain is centered on (0, 0), so this gives uvs in
    // [-0.5, 0.5]. To fix this we add 0.5 to remap to [0, 1]
    let uv = Vec2::new(world_pos.x / dx, world_pos.z / dy);
    uv + 0.5
}
```

![Compute dispatch](/assets/img/andromeda/brush/dispatch_region.png){: width="275" height="275 .right}
With this information, we can determine what part of the heightmap to update. 
Running a compute shader on the entire heightmap would be too costly, not to mention unnecessary.
Instead, we take the center of the brush and only dispatch enough compute invocations to cover a square area
around it. 
The size of this area then corresponds to the brush size. 
A nice additional property of this is that the performance of using a brush doesn't depend on the resolution of the heightmap.
To make the entire process interactive, the commands to execute this compute shader are prepended to the current
frame's rendering commands so the rendered terrain is updated in realtime.

Now we are ready to write the actual compute shader that will modify the heightmap.

```hlsl
// True if the pixel at center + offset is inside the area the brush is allowed to modify
bool inside_patch_rect(int2 center, int2 offset) {
    return abs(offset.x) <= BRUSH_SIZE / 2 && abs(offset.y) <= BRUSH_SIZE / 2;
}

[numthreads(16, 16, 1)]
void main(uint3 GlobalInvocationID : SV_DispatchThreadID) {
    // width and height are the size of the heightmap
    int2 center = int2(float2(width, height) * BRUSH_CENTER_UV);
    int2 offset = int2(GlobalInvocationID.xy) - int(BRUSH_SIZE / 2);
    int2 texel = center + offset;
    // Make sure we do not write out of the image or brush bounds.
    if (texel.x < 0 || texel.y < 0 || texel.x >= width || texel.y >= height) return;
    if (!inside_patch_rect(center, offset)) return;
    // Increase height at the specified location
    float new_height = height_map.Load(int3(texel, 0)) + BRUSH_STRENGTH;
    height_map[texel] = height;
}
```
{: file='height_brush.cs.hlsl'}

And sure enough, when we click somewhere on the terrain mesh, it does rise up from the ground:

![Uniform](/assets/img/andromeda/brush/uniform.png){: width="776" height="563" }

Unfortunately, this is not really how most terrain brush tools work, and it's also quite ugly.
One way to make this feel a lot more natural to use is to change the brush strength based on the 
distance to the center of the brush. 
We already have all the information to do this available in the shader:

```hlsl
float calculate_weight(float distance) {
    // Map the distance to [0, 1] so we can more easily fit a weight function
    float max_distance = BRUSH_SIZE / 2.0;
    float distance_ratio = min(1.0, distance / max_distance);
    return weight_function(distance_ratio);
}

void main() {
  // ...
  float distance = length(float2(offset));
  float weight = calculate_weight(distance);
  float height = heights.Load(int3(texel, 0)) + weight * BRUSH_STRENGTH;
  // ...
}
```
{: file='height_brush.cs.hlsl'}

Now we just need to come up with some good `weight_function` to use. 
After some fiddling in a graphing calculator, I found a good candidate: A gaussian function [^fn-gaussian]. 
The function is defined as follows:

$$g(x) = {1 \over \sigma \sqrt{2 \pi}}exp({-(x - \mu)^2 \over 2\sigma^2})$$

Here the parameter $$\mu$$ is the *mean value* and $$\sigma$$ is the *standard deviation*. 
If you're familiar with statistics, this should be a simple graph to plot for you, but otherwise it's not very obvious
what this looks like.

![gaussian](/assets/img/andromeda/brush/gaussian.png){: width="731" height="548"}
_Gaussian curve with $$\mu = 0$$ and $$\sigma = 0.3$$_

This seems to have some nice properties that we want:
- At $$x = 0$$, the function value is at its peak
- At $$x > 1$$, the function value is very close to zero

Plugging this function as our `weight_function` gives a much nicer result:

![smooth](/assets/img/andromeda/brush/smooth.png){: width="626" height="433"}

### Normal mapping

Unfortunately by introducing this nonuniform brush weight we have created a new problem:
The normal map is now incorrect, since the difference between heights of adjacent pixels has changed.
To fix this, we need to launch another compute shader, this time to recalculate the normal vectors at the points
in and slightly around the height brush.

## Decal rendering

For a nice user experience, it would be nice to render an overlay on the world that shows how the current brush 
will be applied. 
The simplest way to do this is to render a *decal*. 
This is a flat texture that is projected onto 
the rest of the scene geometry. 
There are many ways to render decals, but probably the easiest approach is *Projective Decal Rendering*[^fn-decals]. 
Using this technique, we can avoid generating geometry for the decal that fits the terrain. 
Instead, we render a box that covers the decal area, and project that box onto the terrain using the depth value from
the main rendering pass. 
The drawback of this approach is that it's not vey suitable for rendering many decals, that's where
techniques like *Deferred Decal Rendering* come in.

### Decal box

The box we will render is simply a unit cube scaled with the brush size. 
We do need to pay special attention to the cube's orientation: 
We want our decal space to point up from the terrain, but by default it would be rendered pointing alongside the
default front axis.
In my case, this is the negative Z axis.
To fix this, we simply rotate the cube 90 degrees around the X axis.

```rust
let decal_box_transform = Mat4::from_scale_rotation_translation(
    Vec3::splat(decal.radius),
    Quat::from_rotation_x(90.0f32.to_radians()),
    mouse_world_pos,
);
```

### Decal space

If we want to know the decal UV coordinates, we will need to transform the worldspace coordinates to the *object space*
of the decal. 
We can do this using an orthographic projection matrix and the inverse of the decal box's transformation matrix.

```rust
let to_decal_space = 
    Mat4::orthographic_rh(-0.5, 0.5, -0.5, 0.5, 0.001, 100.0)
    * decal_box_transform.inverse();
```

### Projection

With the CPU-side logic in place, we can now write the shaders needed to render our decal.
The vertex shader is no different from rendering regular meshes: we simply apply the transformation and then the 
projection and view matrices. The real magic happens in the fragment shader. 

```hlsl
float2 decal_uv(float4 frag_pos) {
    float2 frag_uv = frag_pos.xy / float2(rt_width, rt_height);
    float px_depth = depth_rt.SampleLevel(smp, frag_uv, 0).x;
    // Same logic as in the world position reconstruction shader
    float4 world_pos = screen_to_world(frag_uv, px_depth);
    // Transform worldspace position to decal space position
    float3 decal_pos = mul(to_decal_space, world_pos).xyz;
    // Clip any fragments outside our decal
    clip(0.5 - abs(decal_pos));
    // Compute decal uvs from position
    return decal_pos.xy + 0.5;
}
```
{: file='decal.fs.hlsl'}

Here, we compute the decal's UV coordinates, and discard any fragments that are outside the decal.
These UV coordinates can now be used in a similar way as in the height brush shader to display a nice looking brush:

```hlsl
float4 main(PS_INPUT input, float4 frag_pos: SV_Position) : SV_TARGET {
    float2 uv = decal_uv(frag_pos);
    float2 centered_uv = uv * 2.0 - 1.0;
    // Discard everything outside the brush area to make a circle shape
    float distance = length(centered_uv);
    if (distance >= 1.0) {
        return float4(0.0, 0.0, 0.0, 0.0);
    }
    // Use the weight function from before to change the opacity of the decal.
    float weight = weight_function(distance);
    return float4(1.0, 0.0, 0.0, 1.0) * weight;
}
```
{: file='decal.fs.hlsl'}

Which ends up looking like this:

![WithDecal](/assets/img/andromeda/brush/with_decal.png){: width="506" height="354"}
_Decal indicating the brush strength_

With that, all initial goals are achieved! 
To expand on this, a lot of parameters can be made configurable in the editor, such as the brush strength and size,
as well as the weight function to use.
All the source code for this functionality can be found [here](https://github.com/NotAPenguin0/andromeda).

## Video

{% include embed/youtube.html id='sXsCu4l9IX8 ' %}

## Bug gallery

No major undertaking in a graphics project is without some serious bugs, often paired with interesting looking outputs.
I put together a small gallery of images that looked cool or interesting.

![Blocks](/assets/img/andromeda/brush/blocks.png){: width="480" .default}
_That's not exactly smooth_
![Spikes](/assets/img/andromeda/brush/spikes.png){: width="480" .default}
_Applying the brush to a single point isn't very useful_
![stddev](/assets/img/andromeda/brush/stddev.png){: width="480" .default}
_I do not recommend taking $$\sigma = 0.001$$_
![Duplicate](/assets/img/andromeda/brush/duplicate.png){: width="480" .default}
_Two brush decals?_
![Decalbox](/assets/img/andromeda/brush/decal_box1.png){: width="480" .default}
_Those are not correct decal space UVs_
![Decalspace](/assets/img/andromeda/brush/decalspace.png){: width="480" .default}
_Those are also not correct decal space UVs_
![Cutout](/assets/img/andromeda/brush/cutout.png){: width="480" .default}
_Who cut through my decal?_

## Footnotes

[^fn-tess-eval]: Vulkan uses the terminology *Tessellation Control* and *Tessellation Evaluation* shaders, while DX12 uses *Hull* and *Domain* shaders, but they do exactly the same.
[^fn-gaussian]: Also known as a *normal distribution*.
[^fn-decals]: An excellent resource is [Screen Space Decals in Warhammer 40,000: Space Marine](https://blog.popekim.com/en/2012/10/12/siggraph-2012-screen-space-decals-in-space-marine.html)
