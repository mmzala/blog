---
layout: default
title: Ray-Tracing Hair Geometry - Graphics Study
description: Marcin Zalewski - Date TBD
---

Hair geometry rendering has been mostly a mistery for me over the time I have been learning graphics programming.
As so, I have spent the last weeks researching how hair geometry is handled in real-time applications.
Throughout this blog post I'll be sharing what I have learned and some implementations using Vulkan's ray-tracing pipeline.

## Table Of Contents
1. [Introduction To Hair Ray-Tracing](#introduction)
    1. [The Ray-Tracing Pipeline](#ray-tracing-pipeline)
    2. [Defining The Problem](#defining-the-problem)
    3. [Storing Hair Model Data](#stroing-hair-model-data)
2. [Hair Primitives](#hair-primitives)
    1. [DOTS: Disjoint Orthoginal Triangle Strips](#dots)
    2. [The Phantom Ray-Hair Intersector](#prhi) 
    3. [LSS: Linear Swept Spheres](#lss)
    4. [RoCaps: Roving Capsules](#rocaps) 
3. [Conclusion](#conclusion)
   1. [Further Reading](#further-reading)
   2. [Sources](#sources)

## Introduction To Hair Ray-Tracing <a name="introduction"></a>

Ray-tracing, opposite to rasterization, allows us to model our own custom geometry, as rasterization is built around pushing out as many triangles through the pipeline as possible.
This gives us more freedom in how we can represent our data in rendering.

We will leverage this fact for hair rendering to decide from what geometric primitives hair models are made of.

### The Ray-Tracing Pipeline: A Quick Recap <a name="ray-tracing-pipeline"></a>

But before diving into talking about the different primitives we can use, let’s take a quick detour to talk about some important concepts we’ll need to implement our hair geometry.

In ray-tracing we send out rays from the camera’s point of view inside the ray generation shader. These rays then traverse an acceleration structure, to travel through our scene, which in Vulkan’s case is a bounding volume hierarchy that stores instances of our geometry.

<figure align="center" class="image">
<img src="assets/images/hair-geometry/vulkan-rt-pipeline.png" alt="(Vulkan's) Ray-Tracing Pipeline."/>
<figcaption> (Vulkan's) Ray-Tracing Pipeline </figcaption>
</figure>

During the traversal we check using an intersection shader if we intersected with any geometry. When a ray passes this check, an any hit shader is called. This shader is used to tell our pipeline if we want to continue the traversal or discard our ray. This functionality can be used to performs things like alpha-testing to discard intersections.

Finally, the closest hit program is invoked on the intersection closest to the ray origin. It is used to retrieve our geometry information, like positions and normals, and usually used to perform the shading itself. In case our ray doesn’t hit anything, the miss shader is called, which can be used to give the background a color or to render an environment map.

In our case, we are the most interested in the intersection shader, as it allows us to test intersections with our own custom geometry.

### Defining The Problem <a name="defining-the-problem"></a>

So now that we have a basic understanding of the ray-tracing pipeline, we can go back to focus on rendering hair. Let’s first look at what problem we are trying to solve here.

And since we want to render hair models, why not just use textured cards? They are a good way to achieve reasonable quality for hair on a tight frame budget. However, this kind of setup is very limited since a single textured card has multiple hair strands per card which limits control over individual strands. So, by design this approach does not allow much room for simulating or animating hair.

<figure align="center" class="image">
<img src="assets/images/hair-geometry/hair-cards.png" alt="Hair model made out of hair cards"/>
<figcaption> <a href="https://www.ea.com/frostbite/news/frostbite-hair-rendering-and-simulation-2"> Hair model made out of hair cards </a> </figcaption>
</figure>

That is where strand-based approaches come in, where we give each hair strand its own geometry. This would give us finer control over how the hair looks like and give us flexibility to animate and simulate the geometry accurately.

<figure align="center" class="image">
<img src="assets/images/hair-geometry/hair-strands.png" alt="Hair model made out of individual hair strands"/>
<figcaption> <a href="https://www.ea.com/frostbite/news/frostbite-hair-rendering-and-simulation-2"> Hair model made out of individual hair strands </a> </figcaption>
</figure>

But in-turn by using strand-based approaches, we encounter other problems. Like ray-tracing thin geometry that is elongated on an axis, which is horrible for BVH traversal performance. Since elongated geometries have non-optimal bounding volumes with lost of empty space and more overlap between other volumes close by. This results in a higher chance of intersecting the geometry and needing to check more intersections, which we want to avoid if possible.

<figure align="center" class="image">
<img src="assets/images/hair-geometry/bvh-elongated-geometry.png" alt="Hair's BVH leaf nodes visualized, results in elongated thin geometry"/>
<figcaption> Hair's BVH leaf nodes visualized, results in elongated thin geometry </figcaption>
</figure>

There is also the important question of what geometric primitive we want use to represent the individual hair strands. And how many strands do we want to use for hair models? Do we subdivide the hair strands into segments? If so, how many? We’ll take a closer look at these problems throughout this blog post, and I’ll showcase some solutions to them or make them less prominent.

### Storing Hair Model Data <a name="stroing-hair-model-data"></a>

Before rendering any hair model, let’s first look at how we create and store them.

Instead of using triangles, we can use poly lines to approximate the form of a hair strand. Each vertex in a polyline would store a position and a radius, which we would use for the thickness of the hair strand. Such an approximation comes with a cost of quality of course, because we divide the hair strand into segments, we can see that it is not smooth. We could fix such an issue by simply adding more poly line vertices until we think it looks good enough.

<figure align="center" class="image">
<img src="assets/images/hair-geometry/hair-strand-segmented.png" alt="Approximation of a hair strand using polylines"/>
<figcaption> Approximation of a hair strand using polylines </figcaption>
</figure>

But there is also another solution to this problem. We can also use curves to store our hair models. This gets rid of the segmentation entirely, as even when using multiple curves for a hair strand, we can connect them seamlessly. So, it gives us a more realistic result as it correctly follows the path a real hair strand would have. That also means that we can potentially use less curve primitives to create hair compared to polylines, as we no longer need to approximate the path using straight segments.

<figure align="center" class="image">
<img src="assets/images/hair-geometry/hair-strand-curve.png" alt="Accurate hair strand using Bezier curves"/>
<figcaption> Accurate hair strand using Bezier curves </figcaption>
</figure>

## Hair Primitives <a name="hair-primitives"></a>

Now that we understand what data we are working with and what questions we want to answer, we can finally look at ray-tracing hair. 
All the code I show in the following sections are also found on GitHub on my [*vkhrt*](https://github.com/mmzala/vkhrt) repository.
So let's talk rendering the curve geometry.

### DOTS: Disjoint Orthogonal Triangle Strips <a name="dots"></a>

The ray-tracing hardware excels in performing ray-triangle intersections, so why not use them for hair strands? The first technique that does just that, Disjoint Orthogonal Triangle Strips. Or in short DOTS.

<figure align="center" class="image">
<img src="assets/images/hair-geometry/dots-side-view.png" alt="Curve approximated using DOTS geometry - side view"/>
<figcaption> Curve approximated using DOTS geometry - side view </figcaption>
</figure>

<figure align="center" class="image">
<img src="assets/images/hair-geometry/dots-end-view.png" alt="Curve approximated using DOTS geometry - side view"/>
<figcaption> Curve approximated using DOTS geometry - end view </figcaption>
</figure>

DOTS is solution for tessellating curves by using 2 orthogonal quads per segment, which enables viewing from any angle without having to re-orient the triangles to face the camera.
We tessellate such segments by sampling a curved hair strand segment n number of times depending on how much detail we want to create straight line segments.Then we iterate over all the sampled lines to build an orthogonal frame on the line and generate the vertices along both face axes, also taking into account the hair strand radius.

Then we take all these generated triangles and construct a BVH the same way we do as for normal triangle meshes.

This is a pretty good approximation at medium to far distances. But, you may already notice a couple of issues with this approach. For example, how are we going to shade those segments when they are orthogonal to each other?

The quads we use are not naturally round, so we can’t just take the geometry normal to shade the fragments. Although, we can somewhat fix this issue by rounding the geometry normals based on how close the normal is to the edge of the quad. This would give us a better shading normal that approximates how a round primitive that has volume would look like.

<figure align="center" class="image">
<img src="assets/images/hair-geometry/dots-normals.png" alt="Geometry / shading normals"/>
<figcaption> <a href="https://developer.nvidia.com/blog/render-path-traced-hair-in-real-time-with-nvidia-geforce-rtx-50-series-gpus/"> Geometry / shading normals </a> </figcaption>
</figure>

But when we go back to the primitive visualization once more.

Another glaring issue is that we regressed from using curves for smooth hair strands to again approximating them. There are also gaps between each segment, but this is negligible and is only seen when you’re really close to the hair strands.
Both issues can be a bit alleviated by adding more segments to the model when viewing it from medium or far distances but can’t be entirely mitigated at close distances.

<figure align="center" class="image">
<img src="assets/images/hair-geometry/dots-side-view.png" alt="Curve approximated using DOTS geometry - side view"/>
<figcaption> Curve approximated using DOTS geometry - side view </figcaption>
</figure>

<figure align="center" class="image">
<img src="assets/images/hair-geometry/dots-end-view.png" alt="Curve approximated using DOTS geometry - side view"/>
<figcaption> Curve approximated using DOTS geometry - end view </figcaption>
</figure>

So how are we going to solve that?

### The Phantom Ray-Hair Intersector <a name="prhi"></a>

That’s where the Phantom Ray-Hair Intersector comes in.

It allows us to use our curve data directly in our custom intersection shader to check ray-curve intersections, which results in an accurate and smooth hair strand.
The hair strand itself is now also a volume and has accurate normals, which looks much better than our previous method. We also can turn the end caps, which would be flat, on or off.

<figure align="center" class="image">
<img src="assets/images/hair-geometry/prhi-side-view.png" alt="Accurate curve using PRHI geometry - side view"/>
<figcaption> Accurate curve using PRHI geometry - side view </figcaption>
</figure>

<figure align="center" class="image">
<img src="assets/images/hair-geometry/prhi-end-view.png" alt="Accurate curve using PRHI geometry - end view"/>
<figcaption> Accurate curve using PRHI geometry - end view </figcaption>
</figure>

But how do we use our curve data to implement the phantom ray-hair intersector?

#### The Implementation - AABB construction

Now we are working with our own custom curve primitive, which means first we must generate our own acceleration structure that the ray can traverse through.
Luckily Vulkan allows us to feed it axis aligned bounding boxes instead of triangle data, as leaf nodes for BVH construction.

```c++
vk::AccelerationStructureGeometryAabbsDataKHR aabbData {};
aabbData.data = aabbBufferDeviceAddress;
aabbData.stride = sizeof(AABB);

vk::AccelerationStructureGeometryKHR& accelerationStructureGeometry = output.geometry;
accelerationStructureGeometry.flags = vk::GeometryFlagBitsKHR::eOpaque;
accelerationStructureGeometry.geometryType = vk::GeometryTypeKHR::eAabbs;
accelerationStructureGeometry.geometry.aabbs = aabbData; // Instead of triangles, we give AABBs

const uint32_t primitiveCount = hair.aabbCount;

vk::AccelerationStructureBuildRangeInfoKHR& buildRangeInfo = output.info;
buildRangeInfo.primitiveCount = primitiveCount;
buildRangeInfo.primitiveOffset = 0;
buildRangeInfo.firstVertex = 0;
buildRangeInfo.transformOffset = 0;

vk::DeviceOrHostAddressConstKHR curvePrimitiveBufferDeviceAddress {};
curvePrimitiveBufferDeviceAddress.deviceAddress = vulkanContext->GetBufferDeviceAddress(model->curveBuffer->buffer);

GeometryNodeCreation& nodeCreation = output.node;
nodeCreation.primitiveBufferDeviceAddress = curvePrimitiveBufferDeviceAddress.deviceAddress;
nodeCreation.material = hair.material;
```

Then in our intersection shader we can get our curve data using the primitive ID that is the index to the leaf node AABB. Since each curve has its own bounding box,
The same index can be reused to access the curve buffer.

```glsl
void main()
{
    BLASInstance blasInstance = blasInstances[gl_InstanceCustomIndexEXT];
    GeometryNode geometryNode = geometryNodes[blasInstance.firstGeometryIndex + gl_GeometryIndexEXT];

    Curves curves = Curves(geometryNode.primitiveBufferDeviceAddress);
    Curve curve = curves.curves[gl_PrimitiveID];

    // Use curve data for intersection check...
}
```

Because of this, we only need to generate our AABBs for each curve.

If we want a quick solution to approximate a bounding box, we can just include the control points inside the AABB as well.
Although this is a pretty bad solution as it leaves even more empty space in the AABB, so it adds quite a lot of overhead when tracing rays throughout the hair model.

```c++
std::vector<AABB> GenerateAABBs(const std::vector<Curve>& curves, float curveRadius)
{
    std::vector<AABB> aabbs(curves.size());

    for (uint32_t i = 0; i < curves.size(); ++i)
    {
        const Curve& curve = curves[i];
        AABB& aabb = aabbs[i];

        aabb.min = glm::min(glm::min(curve.start, curve.end), glm::min(curve.controlPoint1, curve.controlPoint2)) - curveRadius;
        aabb.max = glm::max(glm::max(curve.start, curve.end), glm::max(curve.controlPoint1, curve.controlPoint2)) + curveRadius;
    }

    return aabbs;
}
```

Instead, we could use a derivative and find the curve’s roots on all axes. This would give us the curve’s extremities. Then if we make sure all these, points together with the start and end point of the curve are all inside the bounding box, it would give us an accurate AABB that tightly bounds the curve.
But as this process can be quite complicated to explain and is math-heavy, it would take some time to explain fully, so I won’t be explaining it in detail. TODO: Add link to source...

<figure align="center" class="image">
<img src="assets/images/hair-geometry/tight-aabb.png" alt="AABB construction from curves"/>
<figcaption> <a href="https://pomax.github.io/bezierinfo/"> AABB construction from curves </a> </figcaption>
</figure>

One of the problems we mentioned at the beginning was that thin long geometry was bad for BVH traversal. We can somewhat overcome this during BVH creation, by generating multiple bounding boxes for curves, which removes a lot of empty space in some instances.

<figure align="center" class="image">
<img src="assets/images/hair-geometry/aabb-leaf-node-split.png" alt="Multiple BVH leaf nodes for each primitive results in better traversal"/>
<figcaption> Multiple BVH leaf nodes for each primitive results in better traversal </figcaption>
</figure>

So now that we have our BVH ready, and we can traverse through it, we want to iteratively find the ray-curve intersection. So, let’s take a look at how we can do that.

#### Phantom Ray-Hair Intersector - The Implementation

We’ll start our iterations at t 0 or 1, where t is the distance along the curve. Each iteration has 3 main steps we have to perform.

<figure align="center" class="image">
<img src="assets/images/hair-geometry/prhi-3-itrs.png" alt="3 iterations of the PRHI algorithm"/>
<figcaption> <a href="https://research.nvidia.com/sites/default/files/pubs/2018-08_Phantom-Ray-Hair-Intersector//Phantom-HPG%202018.pdf"> 3 iterations of the PRHI algorithm </a> </figcaption>
</figure>


1. We first create a cone at distance t along the curve that has the same radius as the thickness of our hair strand. The cone’s axis will travel straight toward the direction we’re trying to converge to.

2. Next we check for the ray-cone intersection. If we intersected the cone and the cone’s start point is close enough to the projected intersection point onto the cone’s base, we can report the intersection and extract the needed information like the position, normals etc. We need to do the second check, where project the intersection point as well, since otherwise we could report an intersection with the cone that would not be on the curve.

3. In case we didn’t find the intersection, we go to step 3, where we use the projected intersection point onto the curve, to travel along it to the next t value. If we are outside of the curve, then we know the ray won’t intersect the curve and we can safely return. If t is still in bounds, we go back to step 1 and iterate until we find an intersection or reach the max iteration count.

After running this algorithm, we’ll have rendered our curve.

#### Phantom Ray-Hair Intersector - The Downsides

But of course, there are downsides. Some curves are impossible to render. For example, if you have a curve that wraps around too much, the real intersection can be hidden by phantom ones, and the ray will never hit the curve as seen on the left picture.
Although that can be fixed by pre-processing the curves and splitting them whenever they wrap around too much.

Another example of an impossible curve is when the start and end radii differ too much, although such as cases are unlikely to happen in real hair models.

<figure align="center" class="image">
<img src="assets/images/hair-geometry/prhi-impossible-curves.png" alt="Impossible curves"/>
<figcaption> <a href="https://research.nvidia.com/sites/default/files/pubs/2018-08_Phantom-Ray-Hair-Intersector//Phantom-HPG%202018.pdf"> Impossible curves </a> </figcaption>
</figure>

There is another big downside to this algorithm. The phantom ray hair intersector runs on average 1.5 times slower than our previous technique where we used triangles.
Due to the nature of it being an iterative approach, the time to intersect with a curve is not constant, so in some cases it may run even slower.

Or at least on my implementation… There are some micro-optimizations I could implement to make the iterative process faster, or optimizations I have not applied to traversing the acceleration structure. But first let’s keep looking if we can find a technique that doesn’t sacrifice as much frame time for quality.

### LSS: Linear Swept Spheres <a name="lss"></a>

Last year, in 2025, NVIDIA released a new ray-tracing primitive with hardware support, Linear Swept Spheres. One of the main applications of the LSS primitive is rendering hair geometry.

<figure align="center" class="image">
<img src="assets/images/hair-geometry/lss-side-view.png" alt="Curve approximated using LSS geometry - side view without end caps"/>
<figcaption> Curve approximated using LSS geometry - side view without end caps </figcaption>
</figure>

<figure align="center" class="image">
<img src="assets/images/hair-geometry/lss-end-view.png" alt="Curve approximated using LSS geometry - end view without end caps"/>
<figcaption> Curve approximated using LSS geometry - end view without end caps </figcaption>
</figure>

<figure align="center" class="image">
<img src="assets/images/hair-geometry/lss-side-view-end-caps.png" alt="Curve approximated using LSS geometry - side view with end caps"/>
<figcaption> Curve approximated using LSS geometry - side view with end caps </figcaption>
</figure>

<figure align="center" class="image">
<img src="assets/images/hair-geometry/lss-end-view-end-caps.png" alt="Curve approximated using LSS geometry - end view with end caps"/>
<figcaption> Curve approximated using LSS geometry - end view with end caps </figcaption>
</figure>

The primitive is a round 3D line with varying radii shaped like a cylinder or cone with optional spheres used as end caps. By chaining them together one after the other we can build curves to represent our hair strands.
Since it is hardware accelerated and the intersection time is constant, it is really fast and runs around the same speed as our first solution that used triangles, but compared to them, it drastically improves memory usage, as we only need to store a position and radius per vertex, where each line has 2 vertices.

But as you may have noticed the curve has been segmented again into sections, which makes the hair strand no longer smooth. And another big limitation is that since its hardware accelerated, it only works on NVIDIA 50xx series GPUs.
So, let’s see if we can do something about these 2 issues.

### RoCaps: Roving Capsules <a name="rocaps"></a>

And we’re in luck, as the LSS primitive from NVIDIA is based off of a paper from 2024 which proposes that hair strands can be modeled by sweeping spheres with varying radii along curves.

<figure align="center" class="image">
<img src="assets/images/hair-geometry/rc-side-view.png" alt="Accurate curve using RoCaps geometry - side view"/>
<figcaption> <a href="https://www.shadertoy.com/view/4ffXWs"> Accurate curve using RoCaps geometry - side view </a> </figcaption>
</figure>

<figure align="center" class="image">
<img src="assets/images/hair-geometry/rc-end-view.png" alt="Accurate curve using RoCaps geometry - end view"/>
<figcaption> <a href="https://www.shadertoy.com/view/4ffXWs"> Accurate curve using RoCaps geometry - end view </a> </figcaption>
</figure>

Such shapes can be ray-traced by finding intersections of a given ray with a set of roving capsules defined at runtime. Or RoCaps in short.
It is also an iterative approach, but it gets a substantial performance boost by eliminating parts of the curved shape that are guaranteed not to intersect with a given ray.
This results in an improvement of ~30% overall in performance over the phantom ray-hair intersector.

#### RoCaps - The Implementation

First, just like for the Phantom Ray-Hair Intersector, we have to generate our own leaf nodes for the BVH. The process is identical to the one described in the PRHI section, so I'll skip it.

The RoCaps algorithm works by finding possible intervals where the ray could intersect by iteratively eliminating parts that are guaranteed not to intersect. This is the most interesting part and I wouldn't be able to do it justice as unfortunately, I have not had the chance yet to implement the algorithm myself. So I really recommend reading the paper for more details about this step!

<figure align="center" class="image">
<img src="assets/images/hair-geometry/rc-intersection-points.png" alt="Intersection intervals elimination"/>
<figcaption> <a href="https://www.researchgate.net/publication/381317645_Modeling_Hair_Strands_with_Roving_Capsules"> Intersection intervals elimination </a> </figcaption>
</figure>

Then for each interval, we make it smaller by checking if a sphere bounding the interval intersects with the ray. If it does, we know we can still intersect with the curve. But if it doesn’t we approximate the root of the interval on the curve. This either makes the interval smaller and deletes it entirely.

If the interval is still there, we compute a capsule also taking into consideration the radius of the curve at the current point and try to intersect with it.
We repeat this process until we intersect or the interval disappears. And due to finding possible intersection intervals at the beginning, an intersection is usually found within 1 or 2 iterations.

#### RoCaps - The Downsides

This technique has many advantages, but of course it comes with limitations and downsides as well. 

Using quadratic or cubic Bézier curves is pretty much a must, adding more control points results in more complex root approximation and is not worth it for performance.

Also, when a curve is quickly changing along with its radius, it can lead to the curve folding onto itself. But again, such a scenario is very unlikely to happen with a hair model.

And RoCaps is still an iterative approach, which means non-linear execution time that may vary per model.
So we again sacrifice performance for quality. But even with these downslides, it is still an overall improvement as even the problems we had with our phantom ray-hair intersector are gone.

<figure align="center" class="image">
<img src="assets/images/hair-geometry/prhi-impossible-curves.png" alt="Impossible curves"/>
<figcaption> <a href="https://research.nvidia.com/sites/default/files/pubs/2018-08_Phantom-Ray-Hair-Intersector//Phantom-HPG%202018.pdf"> 'Impossible' curves are no longer impossible! </a> </figcaption>
</figure>

#### RoCaps - Comparing to LSS

Now for those curious and an important question to ask. If RoCaps exist, then why did NVIDIA implement the algorithm in linear segments? I have had the chance to talk to an NVIDIA researcher that worked on the technology and ask him about it.

The main reason is that since LSS is on a linear segment as that allows for a non-iterative implementation.

This is better for a hardware port as non-iterative implementations have a predictable execution flow, which means you can make assumptions and optimize for it. For example, is known what the exact resource usage will be and it can be accounted for. It is also easier to implement as the sequential flow of logic gates and registers is easier to verify and is less prone to errors.

### Bonus: Voxels? <a name="voxels"></a>

This is a bonus section and can be skipped as I'll be talking about an experiment I did with voxels, which has not been proven in any production application yet.
But if you're interested in voxel rendering, this is the section for you!

My inspiration for using voxels as hair primitives came from the [Witcher 4 demo](https://www.youtube.com/watch?v=Nthv4xF_zHU), where Unreal Engine showcased the ability to render dense thin foliage geometry approximated by voxels.
They make extensive use of the nanite pipeline to convert sub-pixel triangle clusters into voxel clusters stored as 4x4x4 voxel bricks represented as uint64.
These bricks are then traced against during the nanite rasterization pass using a front to back approach, where these clusters are sorted into depth buckets, for early depth testing.

![img.png](assets/images/hair-geometry/nanite-voxels.png)

But as using Unreal Engine 5 and incorporating it into the nanite pipeline would be really time-consuming and is a whole project on its own, I have decided to incorporate voxels into my hair ray-tracer and see some faster results.

#### Voxelization

One of the problems that needed tackling was voxelization of curved segments. While searching the internet I have found an interesting approach for voxelizing 3D lines in a not so old paper, [*Real-Time Rendering of Dynamic Line Sets using Voxel Ray Tracing, 2025*](https://arxiv.org/pdf/2510.09081).
It describes an approach of traversing voxels on the largest axis of a line segment to mark voxels as filled or not filled based on intersecting bounding boxes.

![img.png](assets/images/hair-geometry/voxelization.png)

As voxels are already an approximation of our hair strands, I decided to just segment the curves into smaller straight segments that were 'good enough' and run them through the algorithm shown above.

In my case this approach for voxel generation has been good enough for models I have tested it on.
But you may also want a non-completely-conservative vocalization of your models. The same worry brought me to a paper from 1998, [*An accurate method for voxelizing polygon meshes*](https://dl.acm.org/doi/pdf/10.1145/288126.288181), 
which describes both n-adjacent voxel neighbours checks and n-separating checks to control the conservativeness of the voxelization.

#### Rendering The Voxels

The generated voxels I have stored in a two level voxel brick-map, where each brick stored 4x4x4 voxels represented as an uint64.
Then on top of that, bricks were grouped together into larger bricks for faster traversal throughout the acceleration structure. 
This custom structure I have instanced as a Vulkan BLAS into our scene and used an intersection shader when the BLAS was hit by the hardware accelerated BVH traversal to reverse my own structure.
This way I have leveraged both hardware acceleration and the flexibility for custom geometry.

![img.png](assets/images/hair-geometry/voxel-hair.png)

This is how far I have come for now with voxelized hair. It is not done yet and I have many more ideas how to improve it, but due to time shortage I had to cut the development short for now.

One of my ideas was to add dynamic LODs based on the distance to the camera, which would keep the quality of the hair in check and make sure the voxels would not become much larger than just a pixel.

Another idea was to smooth the normals for better shading, which would improve the illusion of normal hair strands further.
I've found solutions to smoothing voxel normals and can be found in papers such as [*Constrained Elastic Surface Nets: Generating Smooth
Surfaces from Binary Segmented Data*](https://scispace.com/pdf/constrained-elastic-surface-nets-generating-smooth-surfaces-l3ev5gauvh.pdf).

There is also a glaring issue with my implementation. How would you animate voxels? From what I know there aren't any 'good' solutions to this yet.
But it is a topic I am progressively more and more interested in, and I'm considering to look further into voxel animations during my masters.

## Conclusion <a name="conclusion"></a>

So, in the end what technique should you use? That depends...

You should ask yourself what you are trying to use it for. Curly hair using linear primitives will require much more detail, while curved primitives can easily represent such models.
Although curved primitives require iterative approaches to resolve their intersections and will be slower to run. There is also the question of what you need to support and the memory usage of each technique.

I have compiled results from a short test seen below. Everything was tested on a RTX 5070 Ti GPU.

<table style="width:100%; border-collapse: collapse; text-align: center;">
  <thead>
    <tr>
      <th style="border: 1px solid #ccc; padding: 8px;">Hairstyle Model</th>
      <th style="border: 1px solid #ccc; padding: 8px;">Performance Comparison</th>
    </tr>
  </thead>
  <tbody>

 <tr>
   <td style="border: 1px solid #ccc; padding: 8px; width: 40%;">
     <img src="assets/images/hair-geometry/curly-hair.png" alt="Hairstyle A" style="max-width: 100%; height: auto;">
     <div style="margin-top: 6px; font-size: 0.9em;">Curly Hairstyle (11.836.704 curves)</div>
   </td>
   <td style="border: 1px solid #ccc; padding: 8px; width: 60%;">

  <table style="width:100%; border-collapse: collapse;">
    <thead>
      <tr>
        <th style="border-bottom: 1px solid #aaa; padding: 6px;">Technique</th>
        <th style="border-bottom: 1px solid #aaa; padding: 6px;">Frame Time (ms)</th>
        <th style="border-bottom: 1px solid #aaa; padding: 6px;">Memory (MB)</th>
        <th style="border-bottom: 1px solid #aaa; padding: 6px;">Primitives</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td style="padding: 6px;"><b>LSS</b></td>
        <td style="padding: 6px;">~0.30</td>
        <td style="padding: 6px;">379</td>
        <td style="padding: 6px;">11,836,704 linear segments</td>
      </tr>
      <tr>
        <td style="padding: 6px;"><b>DOTS</b></td>
        <td style="padding: 6px;">~0.33</td>
        <td style="padding: 6px;">3,409</td>
        <td style="padding: 6px;">142,040,448 vertices</td>
      </tr>
      <tr>
        <td style="padding: 6px;"><b>PRHI</b></td>
        <td style="padding: 6px;">~1.2</td>
        <td style="padding: 6px;">662</td>
        <td style="padding: 6px;">11,836,704 curves</td>
      </tr>
    </tbody>
  </table>

   </td>
 </tr>

<tr>
   <td style="border: 1px solid #ccc; padding: 8px; width: 40%;">
     <img src="assets/images/hair-geometry/straight-ponytail-hair.png" alt="Hairstyle A" style="max-width: 100%; height: auto;">
     <div style="margin-top: 6px; font-size: 0.9em;">Straight Ponytail Hairstyle (1.656.005 curves)</div>
   </td>
   <td style="border: 1px solid #ccc; padding: 8px; width: 60%;">

  <table style="width:100%; border-collapse: collapse;">
    <thead>
      <tr>
        <th style="border-bottom: 1px solid #aaa; padding: 6px;">Technique</th>
        <th style="border-bottom: 1px solid #aaa; padding: 6px;">Frame Time (ms)</th>
        <th style="border-bottom: 1px solid #aaa; padding: 6px;">Memory (MB)</th>
        <th style="border-bottom: 1px solid #aaa; padding: 6px;">Primitives</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td style="padding: 6px;"><b>LSS</b></td>
        <td style="padding: 6px;">~0.20</td>
        <td style="padding: 6px;">53</td>
        <td style="padding: 6px;">1,656,005 linear segments</td>
      </tr>
      <tr>
        <td style="padding: 6px;"><b>DOTS</b></td>
        <td style="padding: 6px;">~0.265</td>
        <td style="padding: 6px;">477</td>
        <td style="padding: 6px;">19,872,060 vertices</td>
      </tr>
      <tr>
        <td style="padding: 6px;"><b>PRHI</b></td>
        <td style="padding: 6px;">~0.7</td>
        <td style="padding: 6px;">93</td>
        <td style="padding: 6px;">1,656,005 curves</td>
      </tr>
    </tbody>
  </table>

   </td>
 </tr>

  </tbody>
</table>

Unfortunately, I couldn't test RoCaps as I have not had the chance to implement it yet.
It is definitely the next technique I would work on to expand the comparisons between these geometric hair primitives.

In the table above we can see that if you have an RTX 5070 Ti, LSS currently wins by a long shot both in memory and speed.
But you could say that I have given both DOTS and LSS techniques an advantage here, as I have not implemented every idea
I had to improve the Phantom Ray-Hair Intersector. Examples are using fewer curves while maintaining the same detail
and constructing multiple leaf nodes for each curve primitive. This would improve both the frame time and memory usage.
So there is definitely more to explore in terms of improvements as well as things I have not discovered yet.

### Further Reading <a name="further-reading"></a>

If you want to read more about the latest ways to render hair, I would definitely recommend to read the RoCaps paper written by Alexander Reshetov (same author as PRHI) and David Hart.
It gives insights into lessons learned from the Phantom Ray-Hair Intersector and applies them into its spiritual successor, Roving Capsules.

[*Alexander Reshetov and David Hart. Modeling Hair Strands with Roving Capsules, 2024.*](https://www.researchgate.net/publication/381317645_Modeling_Hair_Strands_with_Roving_Capsules)

For people with interest in more math, in 2022 there has been also an interesting talk about polynomial root finding that has also a lot to do with how both Phantom Ray-Hair Intersector and RoCaps work.

[*Cem Yuksel. High Performance Polynomial Root Finding for Graphics - HPG 2022.*](https://www.youtube.com/watch?v=6u8-QFrDY10)

And if you're interested in more implementation details, I would recommend checking out both my and NVIDIA's GitHub repositories. As well as a shader-toy example for RoCaps.

[*vkhrt - My repository on GitHub that includes all the code shown here and more!*](https://github.com/mmzala/vkhrt)

[*NVIDIA. RTX Character Rendering GitHub repository*](https://github.com/NVIDIA-RTX/RTXCR)

[*Alexander Reshetov and David Hart. Roving Capsules Shadertoy Demonstration, 2024*](https://www.shadertoy.com/view/4ffXWs)

### Sources <a name="sources"></a>

- [1] [*Alexander Reshetov and David Luebke. Phantom Ray-Hair Intersector, 2018*](https://research.nvidia.com/sites/default/files/pubs/2018-08_Phantom-Ray-Hair-Intersector//Phantom-HPG%202018.pdf)
- [2] [*Alexander Reshetov and David Hart. Modeling Hair Strands with Roving Capsules, 2024*](https://www.researchgate.net/publication/381317645_Modeling_Hair_Strands_with_Roving_Capsules)
- [3] [*Alexander Reshetov and David Hart. Roving Capsules Shadertoy Demonstration, 2024*](https://www.shadertoy.com/view/4ffXWs)
- [4] [*David Hart and Pawel Kozlowski. Render Path-Traced Hair in Real Time with NVIDIA GeForce RTX 50 Series GPUs, 2025*](https://developer.nvidia.com/blog/render-path-traced-hair-in-real-time-with-nvidia-geforce-rtx-50-series-gpus/)
- [5] [*Mike "Pomax" Kamermans. A Primer on Bézier Curves, 2011-2020*](https://pomax.github.io/bezierinfo/)
- [6] [*Juha Sjoholm. Best Practices for Using NVIDIA RTX Ray Tracing (Updated), 2022*](https://developer.nvidia.com/blog/best-practices-for-using-nvidia-rtx-ray-tracing-updated/)
- [7] [*NVIDIA. RTX Character Rendering GitHub repository*](https://github.com/NVIDIA-RTX/RTXCR)
- [8] [*The Witcher 4 — Unreal Engine 5 Tech Demo*](https://www.youtube.com/watch?v=Nthv4xF_zHU)
- [9] [*Large Scale Animated Foliage in The Witcher 4 Unreal Engine 5 Tech Demo - Unreal Fest Stockholm 2025*](https://www.youtube.com/watch?v=EdNkm0ezP0o)
- [10] [*Hair20K - A Large 3D Hairstyle Database*](https://zhouyisjtu.github.io/project_hair/hair20k.html)

---
*If you made it this far I would like to thank you for spending your time to read my blog post. Feel free to reach out through my e-mail (marcinzal24@gmail.com) with any questions or comments. You can also find me on LinkedIn if you would rather reach out to me that way.*