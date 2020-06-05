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
Skeletal animation is the idea of creating a hierarchical relationship between different shapes, so that when a parent shape moves in some way, it's children move with respect to it.  Additionally, the children are free to move themselves, as long as they respect the movement of their parents. Just like a real skeleton, when I move my arm, my hand moves with it, and as my arm moves, I can make a fist, or rotate my wrist, etc.


I will designate shape 1 as the parent, and shape 2 as it's child.

So, what must happen is we must bring shape 2 into the coordinate space of shape 1, so that shape 1 acts as shape 2's origin.  This way, shape 2 is free to move independently, while still respecting the movement of it's parent. This is accomplished by applying the transformation of shape 1 to shape 2's transformation.

Matrix multiplication is like a mechanism that converts between coordinate spaces.  By multiplying the transform of shape 2 by the transform of shape 1, we end up in shape 1's coordinate space.

In a language like C++, matrix multiplication happens in the reverse of how it's written. The expression `matA * matB` really means "apply MatrixA to MatrixB".

Now, here's all that in code:

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

There! Now shape 2 behaves relative to it's parent. Now this needs be extended to support an arbitrary number of nodes in an arbitrary number of configurations.

If shape 2 had a child, what might it's transform be?  Well, we could reuse `calcShape2TransformMatrix` above, but we need to remember that shape 2's transform is influenced by it's parent! 


## Step 4:
In the general case, a node has arbitrary children, each with their own arbitrary children, on and on. 
A more general implementation might recurse to the leaf nodes, accumulating parent transformations along the way.

```c++
struct PoseableNode {
    std::vector<PoseableNode *> children;
    Transform transform;
    mat4 model; // accumulated transformations
}
```

As we recurse, we save the resulting transformation to `node->model`, so it can be drawn later.
```c++
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

## Step 4.5
Skeletal animation wouldn't be feature complete if we didn't consider the pivot points of individual nodes. Often, when a shape rotates about some axis, it doesn't always rotate about its center of mass. For example, your fingers rotate about your knuckles, and your lower arms rotate about your elbows.  So, for each node, we need to define where it pivots.

For me, this part is what made my brain work the hardest.  Rotating about a pivot works like this:
1. Transform the node so that it's center is on the pivot.
2. Perform the rotation.
3. Apply the inverse of the transform from step 1, so that the translation is nullified, but the rotation stays.

```c++
struct PoseableNode {
    std::vector<PoseableNode *> children;
    Transform transform;
    mat4 model; // accumulated transformations
    vec3 pivot; // relative to node's local origin
}

mat4 revisedCalcLocalTransform(PoseableNode *node) {
    /// everything else is the same
    mat4 pivotTransform = translate(glm::mat4(1.0), node->pivot);
    return translation * inverse(pivotTransform) * rotation * pivotTransform;
}

```

## Step 5
Now, in order to implement a cool animation, we just need to define our skeleton, and specify how a node's transform should change in the update loop.

```c++

/// in main
PoseableNode rightUpperLeg;
rightUpperleg.transform.translation = vec3(-.25, -1.0, 0.0);
rightUpperleg.transfrom.rotation = vec3(0.0, 0.0, -5.0);
rightUpperLeg.transform.scale = vec3(.25, 1.0, .25);

 // make the right upper leg pivot where it meets the torso.
rightUpperLeg.pivot = vec3(0.0, -.5, 0.0);

PoseableNode torso; 
torso.transform.translation = vec3(0.0);
torso.transform.rotation = vec3(0.0);
torso.transform.scale = vec3(.8, 1.0, .3);
torso.children = {&rightUpperLeg ...};

/// in update loop
torso.transform.rotation.y += .1;
rightUpperLeg.transform.rotation.x = abs(cos(current_time)) * 90;

```

### But what about scale?
I arbitrarily decided that I didn't want a node's scale influencing the scale of it's children.  Therefore, I do not consider scale when calculating a node's localTransform - it would end up accumulated. Instead, I consider it later, as I'm about to draw each node.

```c++
void drawNode(PoseableNode *node) {
    mat4 modelWithScale = node->model * scale(mat4(1.0), node->transform.scale);
    drawMesh(modelWithScale);
    for (PoseableNode *child : node->children) {
        drawNode(child);
    }
}
```
