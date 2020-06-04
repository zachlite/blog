---
layout: post
title:  "Skeletal Animation from Scratch"
date:   2020-06-01 19:10:47 -0400
---


In this post, I break down how I implemented a basic, yet functional skeletal animation system.

Here's the end result:





## Step 1:
The simplest possible scenario is just a basic shape whose transform changes over time.

[Video 1]

So far, so good.



## Step 2:
I'm ready to introduce more complexity with a second shape. Just introducing a new shape is obviously not enough, because the second shape is unconcerned with the movement of the first shape.

[video 2]

## Step 3:
Skeletal animation is the idea of creating a hierarchical relationship between different shapes, so that when a parent shape moves in some way, the children move with respect to the parent.  Additionally, the children are free to move themselves, so long as they respect the movement of their parents. Just like a real skeleton, when I move my arm, my hand moves with it, and as my arm moves, I can make a fist, or rotate my wrist, etc.


I will designate shape 1 as the parent, and shape 2 as it's child.

So, what must happen is we must bring shape 2 into the coordinate space of shape 1, so that shape 1 acts as shape 2's origin.  This way, shape 2 is free to move independently, while still respecting the movement of it's parent.



This is accomplished by calculating the transform for Shape2, and applying Shape1's transformation.

Matrix multiplication is like a function that converts one coordinate space to another.  By multiplying the transform of shape2 by the transform of shape1, we end up in shape1's coordinate space.

Don't forget that in a language like C++, matrix multiplication happens in the reverse of how it's written. The expression `matA * matB` really means "apply MatrixA to MatrixB".

```c++

using namespace glm;

// typedefs omitted for brevity
struct Transform {
    vec3 translation;
    vec3 rotation;
    vec3 scale;
}

mat4 calcShape2TransformMatrix(mat4 *shape1TransformMatrix, Transform *s2) {
    // calculate shape2's transformation
    vec3 x = vec3(1.0, 0.0, 0.0);
    mat4 s2Rotation = rotate(mat4(1.0), radians(s2->rotation.x), x);
    mat4 s2Translation = translate(mat4(1.0), s2->translation);
    mat4 s2Transform = s2Translation * s2Rotation;

    // now, shape2 is in the coordinate space of shape1.
    return *shape1TransformMatrix * s2Transform;
}
```

Now we're getting somewhere.





Working on a project that involves posing an animating geometries.
So, I needed to figure out how skeletal animation works and implement it.
this is a place of discovery and learning, so third party implementations are of no use to me.

List out the steps??
Disclaimer:  I had previous knowledge on the subject - not a total noob.

As one should do when they set out to solve a challenging problem:
So the first thing I did was think about simplest possible scenario.  The simplest possible scenario is just a boring piece of geometry whose transform changes over time.

Once that is out of the way, I'm ready to introduce more complexity with a second piece of geometry. Simply introducing new geometry is obviously not enough, because the second shape is unconcerned with the movement of the first shape.

The idea of creating a "skeleton" is that it's a hierarchical relationship between different shapes. If we designate shape 1 as the parent, and shape 2 as it's child, we should expect shape 2 to behave _relative_ to its parent.  When I wave my hand across my desk, the fingers attached to my hand move with it, and yet the are all still free to move themselves, albeit relative to their parent.


Here's the start of this hiearchical relationship between shapes:
{% highlight c++ %}
// typedefs omitted for brevity
struct Transform {
    vec3 translation;
    vec3 rotation;
    vec3 scale;
}

struct PoseableNode {
    std::vector<PoseableNode *> children; // this is a recursive data structure =]
+    Transform transform;
}
{% endhighlight %}

A node's transform must be considered relative to it's parent's transform.  And because this data structure is recursive, a parent's transform is the accumulation of transforms of all predecessors nodes.  Thought of in real terms, the transform (translation, rotation, scale) of my finger is relative to that of my hand. The transform of my hand is relative to my lower arm, and this recurses until there's nothing higher calling the shots.

If I lift my arm, all of its children move with it.


Calculate node transformations -> draw node

I need to discuss the mechanisms of transform accumulation,
pivot,
and how we go about positioning individual nodes.


Before I write about hiearchies of nodes, I need to write about the simplest case of 2 shapes, where one is transformed relative to the other.

I want shape2's position and rotation to be relative to shape1's position and rotation.  I leave scale as independent.
So, what must happen is we must bring shape2 into the coordinate space of its parent.  In other words, shape1 acts as shape2's origin.

This is accomplished by calculating the transform for Shape2, and applying Shape1's transformation.

Matrix multiplication is like a function that converts one coordinate space to another.  By multiplying the transform of shape2 by the transform of shape1, we end up in shape1's coordinate space.

Don't forget that in a language like C++, matrix multiplication happens in the reverse of how it's written. The expression `matA * matB` really means "apply MatrixA to MatrixB".



There! Now Shape2 behaves relative to it's parent.
Now, this needs be extended to support an arbitrary number of nodes in an arbitrary number of configurations.


### videos:
1) single moving shape
2) a second moving shape
3) shape2 moving relative to shape1
4) arbitrary number of shapes in arbitrary configurations
5) Going further.. simple human.


A more general implementation would recurse to the leaf nodes, accumulating parent transformations along the way.
As we recurse, we save the resulting transformation to `node->model`, so it can be drawn later.

```c++
using namespace glm;

struct PoseableNode {
    std::vector<PoseableNode *> children;
    Transform transform;
    mat4 model;  // where we save the resulting transformation
}

mat4 calcLocalTransform(Transform *transform) {
    vec3 x = vec3(1.0, 0.0, 0.0);
    vec3 y = vec3(0.0, 1.0, 0.0);
    vec3 z = vec3(0.0, 0.0, 1.0);
    mat4 rotation = rotate(mat4(1.0), radians(transform->rotation.x), x);
    rotation = rotate(rotation, radians(transform->rotation.y), y);
    rotation = rotate(rotation, radians(transform->rotation.z), z);
    translation = translate(mat4(1.0), transform->translation);
    return translation * rotation;
}

void setNodeTransform(PoseableNode *node, glm::mat4 *parentTransform) {
    node->model = *parentTransform * calcLocalTransform(&node->transform);
    for(PoseableNode *node : node->children) {
        setNodeTransform(node, &node->model);
    }
}

```

But what about scale??
I arbitrarily decided that I didn't want a node's scale influencing the scale of it's children.  Therefore, I do not consider scale when calculating a node's localTransform. I consider it later, as I'm about to draw each node.