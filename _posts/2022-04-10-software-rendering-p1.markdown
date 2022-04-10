
# Exploring Software Rasterizing Performance, Part 1.


## How fast can we make a software rasterizer?

Are modern multicore CPUs fast enough for software rasterizing to be an appropriate choice for game developers?
Which category of games have performance requirements that a software rasterizer can meet, or even exceed?

I have two reasonably modern multicore CPUs available to me. The first is an i7-9700K with 8 cores running at 3.60 GHz.
The second is an M1 with 8 cores running at 2.06 GHz.


Let's find out!

## Motivation:
It is undeniable that GPUs are really good rasterizers, but are they always necessary? Well, probably not, but I don't know much detail beyond that. In quantitative terms, I'd like to know what, if any, circumstances are appropriate for a software rasterizer, instead of hardware acceleration always being the default.


### Why rasterize in software? Why not use dedicated hardware?
- dependency on dedicated hardware for development and consumption. Despite hardware accelerated graphics becoming ubiquitous, they do cost money. Discrete GPUs can be very expensive, if you can manage to buy one at all.


- simpler programming model. If you don't need to communicate with the GPU, you can guarantee that an entire class of bugs and performance bottlenecks will never exist in your code. If you rasterize on dedicated hardware, at some point you are going to need to call an API that communicates data and instructions to the GPU. Effectively, you're programming a distributed system and that comes with its own set of challenges. Both driver overhead and the latency to communicate with the GPU can significantly eat into your frame budget (reference). To compensate, there's a whole big list of things you can do to reduce the number of CPU and GPU round trips (details). But this comes at the cost of more complex code. 


- less dependencies. Direct X? Vendor lock in.  Vulkan? That's too complicated.  OpenGL!? Sure, but deprecated by Apple. Metal! Vendor lock in.
Let's write a hardware Abstraction Layer!! No.
Let's just write a single renderer C++ and be done.



## How does a software rasterizer work



## What is the performance ceiling?


End




Too much work:


In this series of blog posts, I want to answer the question: How competitive is software rendering to hardware accelerated rendering? What metrics will I use to make the 
comparison? What are my constraints? There's probably a difference between 2D and 3D applications.

Question:
At what point does software rendering become noncompetive with hardware accelerated rendering?
Measured in units of frames per second, apples to apples: 
  - n triangles
  - full raster pipeline: clipping, rasterization, etc.


Hypothesis: There is a point on the Effort vs Performance curve that deems software rendering as the superior choice over hardware accelerated rendering.

Background:
Talk about complexities of writing software for a GPU. brief background on GPU. CPUs are good in 2022. back this up with stats!?

Experiment:
- build a feature complete raster pipeline in software.


Are there any valid use cases for a software renderer? What are they? That depends on how fast a software renderer is? What does fast mean? Frames per second? Triangles per second? Need to make sure the comparison is apples to apples.

I'm not trying to suggest or show that software rendering is faster than hardware accelerated rendering. I am asking: What is the highest benchmark a software renderer can achieve, and when might it be the appropriate choice? It would be cool to have a graph that charts Effort vs Performance.  There's likely some area of the graph that your software's requirements might fall into. This is my hypothesis.


Graphics cards dominate the modern rendering scene. From a performance perspective, this makes a lot of sense, becauase speciailized hardware is good at a specialized task: drawing millions of triangles every frame.  I'm sure there are some benchmark numbers somewhere that can support this claim, but its pretty much established that if you wnat to make high performance graphcis applications, you should be utilizing hardware accelerated graphcis. It would be great to break down what high performance graphics applications means. What are the incentives for using a GPU?


Like everything else in life, choices are not black and white, and things have trade offs.  Graphics cards have serious strengths, and serious weaknesses. Discuss the weaknesses - using a graphics card, your software becomes a distributed system.  But people accept this because of the gains. But are people doing so blindly?  So when should people not accept this? Under what circumstances is a software renderer an appropriate choice? This requires a first-principles approach to figuring out just how much performance we can squeeze out of a software renderer on a modern CPU, and comparing that to the performance requirements of the software you indend to write.

Potentially, there's tremendous upside, because we can write simpler code, and target more users. 


How does a software renderer work? Populate pixel buffer and blit to screen. Is there a better word for blit? what is the definition of blit?





# A goal for my writing skills is to incorporate more sources into my writing instead of making hand-wavy claims.
