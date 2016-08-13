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

Bind a Texture to a framebuffer, and use `blitFramebuffer` ... TODO: test if the setup in webgl2 sample pack is minimal


* Make FBO useful
* post processing freed from rendering pipeline

## 3D Texture

The first thing come to mind is Volumetric effects, like fire, smoke, light rays, realistic fog, etc. 
Now we can bring these features into our WebGL engine. 
Besides these, 3D texture be applied for Medical science data like MRI, CT scans. 
It is very useful when implementing cross section. 
In terms of performance, it can be used to cache light for real-time global illumination. 

WebGL support for 3D texture is as good as the 2D one. We have fast accessing speed and 
native tri-linear interpolation. 

The code for setting up a 3D texture mostly has their 2D texture counterpart. 

| Texture 2D | Texture 3D |
|------------|------------|
|`texImage2D`|`texImage3D`|
|`texSubImage2D`|`texSubImage3D`|
|`copyTexImage2D`|`copyTexImage3D`|
|`compressedTexImage2D`|`compressedTexImage3D`|
|`compressedTexSubImage2D`|`compressedTexSubImage3D`|
|`texStorage2D`|`texStorage3D`|

There are certain things that do not match exactly. 
For example, since we have one more dimension we will have `depth`, `zoffset`, `TEXTURE_WRAP_T`
for 3D texture. Also, the internal formate, format, and type combination is not 100% matched. 
In addition, the sampler in shaders is now `sampler3D` instead of `sampler2D`. 
Here's an example setup: 

```javascript
var texture = gl.createTexture();
gl.activeTexture(gl.TEXTURE0);
gl.bindTexture(gl.TEXTURE_3D, texture);
gl.texParameteri(gl.TEXTURE_3D, gl.TEXTURE_BASE_LEVEL, 0);
gl.texParameteri(gl.TEXTURE_3D, gl.TEXTURE_MAX_LEVEL, Math.log2(SIZE));
gl.texParameteri(gl.TEXTURE_3D, gl.TEXTURE_MIN_FILTER, gl.LINEAR_MIPMAP_LINEAR);
gl.texParameteri(gl.TEXTURE_3D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);

gl.texImage3D(
    gl.TEXTURE_3D,  // target
    0,              // level
    gl.R8,        // internalformat
    SIZE,           // width
    SIZE,           // height
    SIZE,           // depth
    0,              // border
    gl.RED,         // format
    gl.UNSIGNED_BYTE,       // type
    data            // pixel
    );

gl.generateMipmap(gl.TEXTURE_3D);
```


Last but not the least, 2D Texture Array is coming together with 3D Texture feature. 
It has its own sampler: `sampler2DArray`, but it shares the `texImage3D` GL functions. 
Here's an example call: 

```javascript
gl.texImage3D(
    gl.TEXTURE_2D_ARRAY,
    0,
    gl.RGBA,
    IMAGE_SIZE.width,
    IMAGE_SIZE.height,
    NUM_IMAGES,
    0,
    gl.RGBA,
    gl.UNSIGNED_BYTE,
    pixels
);
```



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





## Sampler Objects






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
 