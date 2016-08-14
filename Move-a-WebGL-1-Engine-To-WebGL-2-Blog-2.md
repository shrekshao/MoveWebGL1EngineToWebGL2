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

Setting uniform for shaders is a huge part of engine task. Take the [Cesium Globe](http://cesiumjs.org/Cesium/Build/Apps/CesiumViewer/index.html)
as an example. For regular draw calls, `uniform4fv` is within the top 5 GL functions take the most executing time. 
And sum of all `uniform[i]fv` and `uniformMatrix[i]fv` calls is nearly 2.5% of all execution time. 
That's quite a large percentage. We always have to call them to update uniform values each frame. 
What's more, it can be really annoying that we have to make duplicated uniform calls for one same uniform object shared by several shaders.  

Now Uniform buffer object may bring us a boost on performance allowing us to store blocks of uniforms 
in buffers stored on the GPU, just like vertex/index buffers.  
This can make switching between sets of uniforms faster. 
Additionally, uniform buffers can be shared by multiple programs at the same time. 

That's quite a lot benefits. But with so many improvements, the setup routine is about to 
change a lot. We will have a basic setup example first, and then look at something that 
might need your attention. 

```javascript
var uniformPerSceneLocation = gl.getUniformBlockIndex(program, 'PerScene');
gl.uniformBlockBinding(program, uniformPerSceneLocation, 2);
//...
var material = new Float32Array([
    0.1, 0.0, 0.0,  0.0,
    0.0, 0.5, 0.0,  0.0,
    0.0, 0.0, 0.5,  0.0,
    128.0, 0.0, 0.0, 0.0
]);
var uniformPerSceneBuffer = gl.createBuffer();
gl.bindBuffer(gl.UNIFORM_BUFFER, uniformPerSceneBuffer);
gl.bufferData(gl.UNIFORM_BUFFER, material, gl.STATIC_DRAW);
gl.bindBuffer(gl.UNIFORM_BUFFER, null);
//...
// Render
gl.bindBufferBase(gl.UNIFORM_BUFFER, 2, uniformPerSceneBuffer);
```

The first thing that may confuse you is the layout standard (We will focus on std140 here). You can always find the 
details in [OpenGL ES 3.00 Spec](https://www.khronos.org/registry/gles/specs/3.0/es_spec_3.0.0.pdf) Page 68. 

One thing that I really want to get your notice is 

>when the data member is a three-component vector with components consuming N
basic machine units, the base alignment is 4N

And

>If the member is a structure, the base alignment of the structure is N, where
N is the largest base alignment value of any of its members, and rounded
up to the base alignment of a vec4.

Here's an example: 

```GLSL
struct Material
{
    vec3 ambient;
    vec3 diffuse;
    vec3 specular;
    float shininess;
};

uniform PerScene
{
    Material material;
} u_perScene;
```

```javascript
var material = new Float32Array([
    0.1, 0.0, 0.0,  0.0,
    0.0, 0.5, 0.0,  0.0,
    0.0, 0.0, 0.5,  0.0,
    128.0, 0.0, 0.0, 0.0
]);
```

Here we have `vec3`, so our data should add a `0` for each `vec3` for alignment. 
Also, since we use a struct to wrap our data, it should be rounded to a multiply of `vec4`. 
That's why we have 3 extra zero after the `float shininess`. 




Another concerns is aobut updating Uniform Block. There are several different approaches that can get us there. 
But their performance can vary. 

TODO: http://www.gamedev.net/topic/655969-speed-gluniform-vs-uniform-buffer-objects/

http://stackoverflow.com/questions/38841124/updating-uniform-buffer-data-in-webgl-2



## Sync Objects

Sync objects can be used to synchronize execution between the GL server and the client, which 
gives you more control over GPU by letting you set a fence to inform the GPU to wait until a set of 
GL operations have finished. Sync objects are more efficient than `gl.finish`. 

We can get more accurate benchmarks with sync objects. In addition, applications like image 
manipulation where data of each frame comes from CPU will be benefitted with this degree of 
control. 


From Brandon Jones *"What's coming in WebGL 2"*
>With WebGL today the path from Javascript to GPU to Screen fairly opaque to developers. 
You dispatch draw commands and at some undefined point in the future the results (probably) show up on the screen. 
Sync objects allow the developer to gain a little more insight into when the GPU has completed it's work. 
Using gl.fenceSync, you can place a marker at some point in the GPU command stream 
and then later call gl.clientWaitSync to pause Javascript execution 
until the GPU has completed all commands up to the fence. 
Obviously blocking execution isn't desirable for applications that want to render fast, 
but this can be very beneficial for getting accurate benchmarks. 
It may also possibly be used in the future for synchronizing between workers.

* For benchmarks



## Query Objects

This is very useful when we want to do occulsion testing. 
We can know how many geometries are actually drawn by erforming 
a `gl.ANY_SAMPLES_PASSED` query around a set of draw calls. 
We can get rid of those picking method now. 

Keep in mind that these queries are asychronous. A query's result is never available 
in the same frame the query is issued. 

```javascript
gl.beginQuery(gl.ANY_SAMPLES_PASSED, query);
gl.drawArraysInstanced(gl.TRIANGLES, 0, 3, 2);
gl.endQuery(gl.ANY_SAMPLES_PASSED);
//...
(function tick() {
    if (!gl.getQueryParameter(query, gl.QUERY_RESULT_AVAILABLE)) {
        // A query's result is never available in the same frame
        // the query was issued.  Try in the next frame.
        requestAnimationFrame(tick);
        return;
    }

    var samplesPassed = gl.getQueryParameter(query, gl.QUERY_RESULT);
    gl.deleteQuery(query);
})();
```


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
* Bias



## Textures tiny: 
* ETC2/EAC
* Integer textures
* Non-Power-of-Two Texture
    * 2D Sprite
* Additional Texture Formats
* sRGB
 
 ## Primitive restart