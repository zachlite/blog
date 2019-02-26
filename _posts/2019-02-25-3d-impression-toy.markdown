---
layout: post
title: "My First Nontrivial Graphics Programming Project: Building a 3D Impression Toy in WebGL"
date: 2019-02-25 00:00:00
---





<iframe src='https://gfycat.com/ifr/MasculineThirstyAustraliansilkyterrier' frameborder='0' scrolling='no' allowfullscreen width='640' height='593'></iframe>


Here's a <a href="https://zachlite.github.io/3d-impression-toy">demo.</a>

Here's the <a href="https://github.com/zachlite/3d-impression-toy">source code.</a>


This post assumes some familiarlity with graphics programming, namely coordinate systems, and shaders.  It is not my intention to exclude potentially interested readers.  If you'd like a good resource that covers a lot of the requisite concepts, here's an <a href="https://learnopengl.com/" target="_blank">awesome reference</a>.

<h2>Background</h2>


Recently, I built a 3D impression toy to improve my skills as a graphics programmer.  I've been working hard to level up my graphics programming kung foo by reading a ton online, and following various tutorials, but it was time to dive in, and try something on my own.

What's a 3D impression toy?


Why build one?
- They're cool.
- It's a finishable project. It's small enough to let me focus on concepts I want to master. It's challenging enough to make me think and not be bored.


<h2>How did I do it?</h2>

I broke the problem down into these parts:

1. Basic scene setup
2. Scaling the model to fit in a unit volume (1x1x1)
3. Dividing the xz plane into cells based on some resolution
4. Drawing columns in each of the cells
5. Only drawing the columns up to the height where it pokes through the top of the model

<h2>Basic Scene Setup</h2>
To get a basic scene up and running, I need a model to render, and a some way to render it.

For a model, I use the Stanford bunny, mostly out of convenience, since it exists as an npm module. My implementation allows for any arbitrary model to be used, but I only make use of the bunny for now.  

For drawing, I use the the wonderful regl project to interact with WebGL.  Rather than interacting with WebGL state directly in a super side-effecty way, state changes are managed by regl via a nice declaritive API.

There are no surprises getting this part set up. Here are the highlights:
- Load the bunny's vertex and edge data
- Create a world space transform for the bunny
- Calculate a model matrix using the transform and pass it to the WebGL context via regl
- Use regl-camera to magically create and pass the projection and view matrices to the WebGL context. (3rd party libraries are good when I'm in a 3D Impression toy making hurry!)
- Apply the projection, view, and model matrices to draw the bunny in clip space.

I add some lighting in the fragment shader for realism, and here's my progress so far:
 <!-- include video of just bunny being drawn -->
<img src="/assets/img/bunnyworldspace.png"/>

## Scaling the model to fit in a unit volume

For simplicity (and sanity) in the following steps, I want scale in world space to be universally the same for any 3rd party model. In other words, I want scale in model space to equal 1. This was accomplished by taking every single vertex of the model and dividing each x, y, and z component by the largest x, y, or z.

e.g.: 

{% highlight javascript %}
function normalizeVertices(vertices: number[][]) {
  const max: number = _.max(_.flattenDeep(vertices));
  return vertices.map(pos => pos.map(n => n / max));
}

console.log(normalizeVertices([[1, 2, 3], [0, 5, 0], [2, 0, 8]]));
// [[.125, .25, .375], [0, .625, 0], [.25, 0, 1]]
{% endhighlight %}


## Divide the xz plane into cells
A column moves along the y axis, so a column's position and dimensions are considered in x and z.

Most 3D impression toys have their columns placed in a hexagonal arrangement. For simplicity, I make a grid of square cells that is the same number of cells wide as it is deep.

My goal here is to write a function that returns an array of world space transforms, each representing the position and dimensions of a column that will eventually be drawn.

Important considerations:
- I want to be able to change the cell density of the grid, so the desired cell amount is parameterized.
- Naively, the first cell will be placed at (x = 0, z = 0) and subsequent cells will be placed in the positive x and z directions.  Instead, cell positions need to be offset so that the center of the _grid_ is at (x = 0, z = 0).

Here's an illustration of the grid and bunny in world space to show what the heck I'm talking about.

<img src="/assets/img/cells.png"/>


## Drawing columns in each cell

Now I start drawing columns! 

For a model, I use a cube whose world space y scale has been set to 1, the height of the unit volume.

The cells I defined in the previous step tell me the position and dimensions for each column to be drawn, so I loop over the cells and do a draw call for each one.  I also go back and add a little bit of spacing between cell positions for some extra realism.

At this point, I use a separate vertex and fragment shader for the column, since I'll be writing some very column-specific shader code in the next step.  

This gets me to here:
<div style='position:relative; padding-bottom:calc(92.75% + 44px)'><iframe src='https://gfycat.com/ifr/ImmediateInsidiousGelada' frameborder='0' scrolling='no' width='100%' height='100%' style='position:absolute;top:0;left:0;' allowfullscreen></iframe></div>

## Only drawing columns up to where they intersect the surface of the model

This part is where the real magic happens.
A fragment of the column should only be drawn if that fragment's height in world space is below where the xz center of the column makes its final intersection with the surface of the model.  That's a mouthful.  Here's a picture to show what I mean:

<img src="/assets/img/columnvssurfaceheight.png"/>



<!-- Aside: For those unfamiliar with fragments and fragment shaders, the idea is what we're trying to draw has been split up into many tiny _fragments_ that all need an assigned color, and this shader is responsible for assigning those colors. 1 fragment that needs color = 1 fragment shader invocation. -->

<!-- (include picture to show what this means) -->

So in the column's fragment shader, this is what I'm aiming for:

{% highlight javascript %}
if (thisFragmentHeight <= heightWhereColumnPokesThroughShape) {
  alpha = 1.0; // shade this fragment
} else {
  alpha = 0.0; // don't shade this fragment
}
{% endhighlight %}


Figuring out the fragment's height comes easily, since if I define a varying variable in the vertex shader, it's value will be interpolated for each fragment and accessable to us in the fragment shader.

Now what about the height where the column pokes throught the shape? There's a problem with our approach so far. In world space, finding the height where the column pokes through the model is kind of a nightmare. For each of the model's position vertices, I need to see if it's contained within our column, and if it is, I need to see if there is a position vertex above it, within the xz bounds of the column. When there's no vertex above, the column pokes through. Besides the exponential complexity, I'd need to pack the entire model's position coordinates in a texture and send it to the fragment shader to have these variables in scope to do the calculation.

Yuck.

#### Making a depth map

An alternative approach is to make a depth map of the bunny from a bird's eye view.  I'll refer to this bird's eye view as depth space, and it's the primary coordinate system I'll be working in intead of world space, because it's the perspective used to make the depth map.

A depth map encodes distances from the viewer as greyscale color, so something really far away, at a distance of 1.0 appears as white, while something really close, at a distance of 0.0, appears as black. So, I create a depth map of the bunny in depth space, and when I want to know the distance from the depth map camera to the surface at a specific position, I can sample the color of the depth map at that position. Because distance is encoded as greyscale color, the sampled color's blue component is z.

Here's a depth map of the bunny:

<img src="/assets/img/depthmap.png"/>




In the column's fragment shader, I can sample the depth map at the column's xz center and test whether or not the sampled value is larger or smaller than the fragment's depth.

Remember, I'm working in depth space now, so you can think of a fragment's depth as it's distance from the depth map camera (the bird's eye view from which the depth map was created).

Here's an illustration showing the difference between considering heights in world space and depths in depth space.

<img src="/assets/img/worldvsdepthspace.png"/>

Similarly, the depth encoded in the depth map is also a distance from the depth map camera, so the annoyingly complicated world space problem described earlier turns in to the shockingly simple depth space solution:

{% highlight javascript %}

if (fragmentDepth > surfaceDepth) {
  alpha = 1.0;
} else {
  alpha = 0.0;
}

{% endhighlight %}


When the fragment's depth is farther away from the depth map camera than the surface, the column hasn't poked through yet, so it's alpha should be 1.0.

Conversely, when the fragments's depth is closer to the camera, than the fragment's depth, the column has poked through, and that fragment of the column should have its alpha set to 0.0.



## Thanks for reading!


What did you think of this post?
Could I have done something better?
Do you know something cool about graphics programming that you think I should know?

Feel free to send me an email:
zachlite@gmail.com


