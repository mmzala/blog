---
layout: default
title: Ray-Tracing Hair Geometry - Graphics Study
description: Marcin Zalewski - Date TBD
---

TODO: Small intro...

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

TODO...

### The Ray-Tracing Pipeline: A Quick Recap <a name="ray-tracing-pipeline"></a>

But first let’s take a quick detour to talk about some important concepts we’ll need to implement our hair geometry.

In ray-tracing we send out rays from the camera’s point of view inside the ray generation shader. These rays then traverse an acceleration structure, to travel through our scene, which in Vulkan’s case is a bounding volume hierarchy that stores instances of our geometry.

During the traversal we check using an intersection shader if we intersected with any geometry. When a ray passes this check, an any hit shader is called. This shader is used to tell our pipeline if we want to continue the traversal or discard our ray. This functionality can be used to performs things like alpha-testing to discard intersections.

Finally, the closest hit program is invoked on the intersection closest to the ray origin. It is used to retrieve our geometry information, like positions and normals, and usually used to perform the shading itself. In case our ray doesn’t hit anything, the miss shader is called, which can be used to give the background a color or to render an environment map.

In our case, we are the most interested in the intersection shader, as it allows us to test intersections with our own custom geometry.

### Defining The Problem <a name="defining-the-problem"></a>

So now that we have a basic understanding of the ray-tracing pipeline, we can go back to focus on rendering hair. Let’s first look at what problem we are trying to solve here.

And since we want to render hair models, why not just use textured cards? They are a good way to achieve reasonable quality for hair on a tight frame budget. However, this kind of setup is very limited since a single textured card has multiple hair strands per card which limits control over individual strands. So, by design this approach does not allow much room for simulating or animating hair.

That is where strand-based approaches come in, where we give each hair strand its own geometry. This would give us finer control over how the hair looks like and give us flexibility to animate and simulate the geometry accurately.

--- Next Section ---

But in-turn by using strand-based approaches, we encounter other problems. Like ray-tracing thin geometry that is elongated on an axis, which is horrible for BVH traversal performance. Since elongated geometries have non-optimal bounding volumes with lost of empty space and more overlap between other volumes close by. This results in a higher chance of intersecting the geometry and needing to check more intersections, which we want to avoid if possible.

There is also the important question of what geometric primitive we want use to represent the individual hair strands. And how many strands do we want to use for hair models? Do we subdivide the hair strands into segments? If so, how many? We’ll take a closer look at these problems throughout this presentation, and I’ll showcase some solutions to them or make them less prominent.

### Storing Hair Model Data <a name="stroing-hair-model-data"></a>

Before rendering any hair model, let’s first look at how we create and store them.

Instead of using triangles, we can use poly lines to approximate the form of a hair strand. Each vertex in a polyline would store a position and a radius, which we would use for the thickness of the hair strand. Such an approximation comes with a cost of quality of course, because we divide the hair strand into segments, we can see that it is not smooth. We could fix such an issue by simply adding more poly line vertices until we think it looks good enough. But there is also another solution to this problem.

--- Next Section ---

We can also use curves to store our hair models. This gets rid of the segmentation entirely, as even when using multiple curves for a hair strand, we can connect them seamlessly. So, it gives us a more realistic result as it correctly follows the path a real hair strand would have. That also means that we can potentially use less curve primitives to create hair compared to polylines, as we no longer need to approximate the path using straight segments.


## Hair Primitives <a name="hair-primitives"></a>

TODO...

### DOTS: Disjoint Orthogonal Triangle Strips <a name="dots"></a>

The ray-tracing hardware excels in performing ray-triangle intersections, so why not use them for hair strands? The first technique that does just that, Disjoint Orthogonal Triangle Strips. Or in short DOTS.

DOTS is solution for tessellating curves by using 2 orthogonal quads per segment, which enables viewing from any angle without having to re-orient the triangles to face the camera.
We tessellate such segments by sampling a curved hair strand segment n number of times depending on how much detail we want to create straight line segments.Then we iterate over all the sampled lines to build an orthogonal frame on the line and generate the vertices along both face axes, also taking into account the hair strand radius.

Then we take all these generated triangles and construct a BVH the same way we do as for normal triangle meshes.

This is a pretty good approximation at medium to far distances. But, you may already notice a couple of issues with this approach. For example, how are we going to shade those segments when they are orthogonal to each other?

--- Next Section ---

The quads we use are not naturally round, so we can’t just take the geometry normal to shade the fragments. Although, we can somewhat fix this issue by rounding the geometry normals based on how close the normal is to the edge of the quad. This would give us a better shading normal that approximates how a round primitive that has volume would look like.

--- Next Section ---

When we go back to the primitive visualization.

Another glaring issue is that we regressed from using curves for smooth hair strands to again approximating them. There are also gaps between each segment, but this is negligible and is only seen when you’re really close to the hair strands.
Both issues can be a bit alleviated by adding more segments to the model when viewing it from medium or far distances but can’t be entirely mitigated at close distances.

So how are we going to solve that?

### The Phantom Ray-Hair Intersector <a name="prhi"></a>

TODO...

### LSS: Linear Swept Spheres <a name="lss"></a>

TODO...

### LSS: RoCaps: Roving Capsules <a name="rocaps"></a>

TODO...


## Conclusion <a name="conclusion"></a>

TODO...

### Further Reading <a name="further-reading"></a>

TODO...

### Sources <a name="sources"></a>

- [1] [*Alexander Reshetov and David Luebke. Phantom Ray-Hair Intersector, 2018*](https://research.nvidia.com/sites/default/files/pubs/2018-08_Phantom-Ray-Hair-Intersector//Phantom-HPG%202018.pdf)

---