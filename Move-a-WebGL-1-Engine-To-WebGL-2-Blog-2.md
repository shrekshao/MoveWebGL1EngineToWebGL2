Last time we figure out how to deal with issues porting WebGL 1 engine to WebGL 2. 
In this article, we will talk about what new features come with WebGL 2 and 
what cool things can we do with them in our engine. 

# New features

## Multisampled Renderbuffers

Previously if we want antialising we would either have to 
render it to the default backbuffer or 
perform our own post-process AA (like FXAA) on content rendered to a texture.

But with Multisampled Renderbuffers, we can now use the general renderring pipeline in 
WebGL as other platform. 
> pre-z pass --> rendering pass to FBO --> postprocessing pass --> render to window

`renderbufferStorageMultisample` is the function directly related here. 

```javascript
var colorRenderbuffer = gl.createRenderbuffer();
gl.bindRenderbuffer(gl.RENDERBUFFER, colorRenderbuffer);
gl.renderbufferStorageMultisample(gl.RENDERBUFFER, 4, gl.RGBA8, FRAMEBUFFER_SIZE.x, FRAMEBUFFER_SIZE.y);
```

Bind a Texture to a framebuffer, and use `blitFramebuffer` ... TODO



* Make FBO useful
* post processing freed from rendering pipeline

## 3D Texture
* Volumetric effects in games (fire, smoke, light rays, realistic fog)
* Caching light for real time global illumination
* MRI, CT scans, cross section
* Terrain textures


## Sampler Objects


## Uniform Buffer
* Performance boost (profiling several current engine and see how much they bottleneck at uniform?)
Bound to multiple programs at the same time
* Std140 layout (https://www.khronos.org/registry/gles/specs/3.0/es_spec_3.0.0.pdf  page 68)
* How to use ubo api calls (http://www.gamedev.net/topic/655969-speed-gluniform-vs-uniform-buffer-objects/ )


## Sync Objects
* For benchmarks

## Query Objects
* Occlusion testing
* picking


## Transform Feedback
* Particle system (Simulation)


## New GLSL 3.00 ES Shader

* A list of tiny features (optional)
    - Layout qualifiers
    - Vertex texture
    - Non sqaure matrix
    - flat/smooth interpolators
    - centroid
    - Fragment discard
    - New built-in functions
    - Texture Grad
    - ...


## Texture LOD
* Texture Network streaming


## Primitive restart


## Textures tiny: 
* ETC2/EAC
* Integer textures
* Non-Power-of-Two Texture
* 2D Sprite
* Additional Texture Formats
* sRGB
 