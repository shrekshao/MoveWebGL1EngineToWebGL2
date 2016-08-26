Last time we showed how to deal with issues porting a WebGL 1 engine to WebGL 2. 
In this article, we will talk about what new features come with WebGL 2 and 
what cool things can we do with them. 

# New features

## Multisampled Renderbuffers

Previously, if we want antialiasing we would either have to 
render it to the default backbuffer or 
perform our own post-process AA (such as FXAA or [SMAA](http://threejs.org/examples/#webgl_postprocessing_smaa)) on content rendered to a texture.

Now, with Multisampled Renderbuffers, we can now use the general rendering pipeline in 
WebGL to provide multisampled antialiasing (MSAA):
> pre-z pass --> rendering pass to FBO --> postprocessing pass --> render to window

`renderbufferStorageMultisample` is the relevant function here. 

```javascript
var colorRenderbuffer = gl.createRenderbuffer();
gl.bindRenderbuffer(gl.RENDERBUFFER, colorRenderbuffer);
gl.renderbufferStorageMultisample(gl.RENDERBUFFER, 4, gl.RGBA8, FRAMEBUFFER_SIZE.x, FRAMEBUFFER_SIZE.y);

gl.bindFramebuffer(gl.FRAMEBUFFER, framebuffers[FRAMEBUFFER.RENDERBUFFER]);
gl.framebufferRenderbuffer(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT0, gl.RENDERBUFFER, colorRenderbuffer);
gl.bindFramebuffer(gl.FRAMEBUFFER, null);
```

Pay attention to the fact that the multisample renderbuffers cannot be directly bound to textures, 
but they can be resolved to single-sample textures 
using the `blitFramebuffer` call. This is a 
new feature in WebGL 2 as well, and is used like this: 

```javascript
var texture = gl.createTexture();
gl.bindTexture(gl.TEXTURE_2D, texture);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.NEAREST);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.NEAREST);
gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, FRAMEBUFFER_SIZE.x, FRAMEBUFFER_SIZE.y, 0, gl.RGBA, gl.UNSIGNED_BYTE, null);
gl.bindTexture(gl.TEXTURE_2D, null);

gl.bindFramebuffer(gl.FRAMEBUFFER, framebuffers[FRAMEBUFFER.COLORBUFFER]);
gl.framebufferTexture2D(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT0, gl.TEXTURE_2D, texture, 0);
gl.bindFramebuffer(gl.FRAMEBUFFER, null);

// ...

// After drawing to the multisampled renderbuffers
gl.bindFramebuffer(gl.READ_FRAMEBUFFER, framebuffers[FRAMEBUFFER.RENDERBUFFER]);
gl.bindFramebuffer(gl.DRAW_FRAMEBUFFER, framebuffers[FRAMEBUFFER.COLORBUFFER]);
gl.clearBufferfv(gl.COLOR, 0, [0.0, 0.0, 0.0, 1.0]);
gl.blitFramebuffer(
    0, 0, FRAMEBUFFER_SIZE.x, FRAMEBUFFER_SIZE.y,
    0, 0, FRAMEBUFFER_SIZE.x, FRAMEBUFFER_SIZE.y,
    gl.COLOR_BUFFER_BIT, gl.NEAREST
);
```


## 3D Texture

The first thing that comes to mind with 3D textures is volumetric effects, such as fire, smoke, light rays, realistic fog, etc. 
Now we can bring these features into our WebGL engine. 
In addition, 3D textures can be used to store medical data such as MRI and CT scans, and are useful when implementing cross-sectioning. 
3D textures can also improve performance by using them to cache light for real-time global illumination. 

WebGL 2 support for 3D textures is as good as that for 2D textures. We have fast access speed and 
native tri-linear interpolation. 

The code for setting up a 3D texture usually has a 2D texture counterpart. 

| Texture 2D | Texture 3D |
|------------|------------|
|`texImage2D`|`texImage3D`|
|`texSubImage2D`|`texSubImage3D`|
|`copyTexImage2D`|`copyTexImage3D`|
|`compressedTexImage2D`|`compressedTexImage3D`|
|`compressedTexSubImage2D`|`compressedTexSubImage3D`|
|`texStorage2D`|`texStorage3D`|

There are certain elements that do not match exactly. 
For example, since we have one more dimension, we will have `depth`, `zoffset`, and `TEXTURE_WRAP_T`
for 3D textures. Also, the internal format and type combinations are not 100% matched.

The sampler used in shaders is `sampler3D` instead of `sampler2D`. 

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


Last but not the least, the 2D Texture Array concept is available with the 3D Texture feature. That is, multiple 2D textures can be stored in an array that can be accessed.
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

Setting uniforms for shaders is often a considerable amount of the time spent by an engine. Take the [Cesium Globe](http://cesiumjs.org/Cesium/Build/Apps/CesiumViewer/index.html)
as an example. For regular draw calls, `uniform4fv` is within the top 5 GL functions taking the most execution time. 
Also, the sum of all `uniform[i]fv` and `uniformMatrix[i]fv` calls is nearly 2.5% of all execution time. 
That's quite a large percentage. We always have to call them to update uniform values each frame. 
What's more, it can be annoying that we have to make duplicated uniform calls for one same uniform object shared by several shaders.  

Now the Uniform buffer object may bring us a boost in performance by allowing us to store blocks of uniforms 
in buffers stored on the GPU, just like vertex/index buffers.  
This can make switching between sets of uniforms faster. 
Additionally, uniform buffers can be shared by multiple programs at the same time. 

That's quite a few benefits. But, with so many improvements, the setup routine is about to 
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

The first thing that may confuse you is the layout standard (we will focus on std140 here). You can always find the 
details in [OpenGL ES 3.00 Spec](https://www.khronos.org/registry/gles/specs/3.0/es_spec_3.0.0.pdf) Page 68. 

One thing that I really want you to notice is:

>when the data member is a three-component vector with components consuming N
basic machine units, the base alignment is 4N

And:

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
Also, since we use a struct to wrap our data, it should be rounded to a multiple of `vec4`. 
That's why we have 3 extra zeroes after the `float shininess`. 

Another concern is about updating the uniform block. There are several different approaches that can get us there. 
However, their performance can vary. It's pretty tricky to make the most of our uniform block. 

Here are some detailed discussions on [Stack Overflow](http://stackoverflow.com/questions/38841124/updating-uniform-buffer-data-in-webgl-2.) and [gamedev.net](
http://www.gamedev.net/topic/655969-speed-gluniform-vs-uniform-buffer-objects/). 

But, basically, we can use `gl.bufferSubData` to copy the updated typedArray into the uniform buffers. 


## Sync Objects

Sync objects can be used to synchronize execution between the GL server and the client, which 
gives you more control over GPU by letting you set a fence to inform the GPU to wait until a set of 
GL operations have finished. Sync objects are more efficient than `gl.finish`. 

We can get more accurate benchmarks with sync objects. In addition, applications such as image 
manipulation, where data of each frame comes from the CPU, will benefit from this degree of 
control. 


## Query Objects

This operation is very useful when we want to do occlusion testing. 
We can know how many geometries are actually drawn by performing 
a `gl.ANY_SAMPLES_PASSED` query around a set of draw calls. 
We can use these queries and so get rid of specialized picking method code. 

Keep in mind that these queries are asynchronous. A query's result is never available 
in the same frame that the query is issued. This is different from OpenGL ES 3 where query result 
may be available in the same frame. It's an application portability concern. 

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





## Sampler Objects

In WebGL 1 texture image data and sampling information (which tells GPU how to read the image data) 
are both stored in texture objects. It can be annoying when we want to read from the same texture twice 
but with a different method (say, linear filtering vs nearest filtering) because we need to have 
two texture objects. But, with sampler objects, we can separate these two concepts. We can have one 
texture object and two different sampler objects. This will result in a change in how our engine organize 
textures.  
Here's an example: 

```javascript
var samplerA = gl.createSampler();
gl.samplerParameteri(samplerA, gl.TEXTURE_MIN_FILTER, gl.NEAREST_MIPMAP_NEAREST);
gl.samplerParameteri(samplerA, gl.TEXTURE_MAG_FILTER, gl.NEAREST);
gl.samplerParameteri(samplerA, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
gl.samplerParameteri(samplerA, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);

var samplerB = gl.createSampler();
gl.samplerParameteri(samplerB, gl.TEXTURE_MIN_FILTER, gl.LINEAR_MIPMAP_LINEAR);
gl.samplerParameteri(samplerB, gl.TEXTURE_MAG_FILTER, gl.LINEAR);
gl.samplerParameteri(samplerB, gl.TEXTURE_WRAP_S, gl.MIRRORED_REPEAT);
gl.samplerParameteri(samplerB, gl.TEXTURE_WRAP_T, gl.MIRRORED_REPEAT);

// ...

gl.activeTexture(gl.TEXTURE0);
gl.bindTexture(gl.TEXTURE_2D, texture);
gl.bindSampler(0, samplerA);

gl.activeTexture(gl.TEXTURE1);
gl.bindTexture(gl.TEXTURE_2D, texture);
gl.bindSampler(1, samplerB);
```


## Transform Feedback

Transform feedback allows the output of the vertex shader to be captured in a buffer object. 
This is useful for particle systems and simulation that perform on the GPU without 
any CPU intervention. 

In WebGL 1, when we want to implement such feature, usually a texture 
storing the states of particles is inevitable. Two textures, to be precise, 
storing states from previous frame and current frame, and ping-pong between them.

Here's an example of WebGL 1 approach (from [toji's WebGL Particles take 2](https://github.com/toji/webgl2-particles-2))

In the first pass fragment shader, 
do the simulation, and store the position results in a texture. 

```GLSL
// First pass - Fragment Shader
uniform sampler2D tPositions;
varying vec2 vUv;
// ...
vec4 runSimulation(vec4 pos) {
    // simulation
    // ...
    return pos;
}

void main() {
    vec4 pos = texture2D( tPositions, vUv );
    //...
    pos = runSimulation(pos);

    // Write new position out
    gl_FragColor = pos;
}
```

And then we use this position texture as an input for our second pass vertex shader.

```GLSL
// Second pass - Vertex Shader
attribute vec3 position;
uniform float pointSize;
uniform sampler2D map;
varying vec2 vUv;

//...

void main() {
    vUv = position.xy + vec2( 0.5 / width, 0.5 / height );
    vec3 color = texture2D( map, vUv ).rgb;
    gl_PointSize = pointSize;
    gl_Position = projectionMatrix * modelViewMatrix * vec4( color, 1.0);
}
``` 

This is how we do it in WebGL 2. With Transform feedback, we can discard the fragment shader 
in step 1, as well as the texture. We write the output (position) of the vertex shader in step 1 to the 
vertex attribute array input of step 2. (In practice, you still need a placeholder trivial fragment shader 
for the first step to correctly compile the program.)


```GLSL
// First pass - Vertex Shader
// ...
out vec3 v_position;
void main() {
    // ...
    v_position = u_projMatrix * u_modelViewMatrix * vec4(a_position, 1.0);
}
```

```GLSL
// Second pass - Vertex Shader
in vec3 a_position;
void main() {
    gl_Position = projectionMatrix * modelViewMatrix * vec4( a_position, 1.0 );
    // ...
}
```

And here's how we bind the buffers (from [WebGL2SamplesPack](https://github.com/WebGLSamples/WebGL2Samples)):

```javascript
var transformFeedback = gl.createTransformFeedback();
var varyings = ['v_position', /*...*/];
gl.transformFeedbackVaryings(programTransform, varyings, gl.SEPARATE_ATTRIBS);
// ...
gl.bindBuffer(gl.ARRAY_BUFFER, particleVBOs[i][Particle.POSITION]);
gl.bufferData(gl.ARRAY_BUFFER, particlePositions, gl.STREAM_COPY);
// ...
gl.bindTransformFeedback(gl.TRANSFORM_FEEDBACK, transformFeedback);
gl.bindBufferBase(gl.TRANSFORM_FEEDBACK_BUFFER, 0, particleVBOs[(currentSourceIdx + 1) % 2][Particle.POSITION]);
gl.enable(gl.RASTERIZER_DISCARD);   // we are not drawing
gl.beginTransformFeedback(gl.POINTS);

gl.drawArrays(gl.POINTS, 0, NUM_PARTICLES);

gl.endTransformFeedback();
gl.disable(gl.RASTERIZER_DISCARD);

gl.bindBufferBase(gl.TRANSFORM_FEEDBACK_BUFFER, 0, null);
```


## A set of texture new features:

Here is a list of the new texture features in WebGL 2. 

* sRGB textures
Allow the application to perform gamma-correct rendering. 

```javascript
gl.texImage2D(
    gl.TEXTURE_2D,
    0, // Level of details
    gl.SRGB8, // Format
    gl.RGB,
    gl.UNSIGNED_BYTE, // Size of each channel
    image
);
```

The sRGB texture will be automatically converted to linear space when being fetched in the shader. For physically-based rendering and other operations we normally want to deal with colors in a linear space, not a display space.


* Vertex texture
    * terrain
    * water
    * skeleton animation

* Texture LOD 

The texture LOD parameter
is used to determine which mipmap to fetch from; it can now be
clamped. The base and maximum mipmap level can both
be set as clamps. This allows mipmap streaming, i.e., loading only the mipmap levels currently needed. This is very useful for a WebGL environment, 
where textures are downloaded via a network. 

```javascript
gl.texParameterf(gl.TEXTURE_2D, gl.TEXTURE_MIN_LOD, 0.0);
gl.texParameterf(gl.TEXTURE_2D, gl.TEXTURE_MAX_LOD, 10.0);
```

* ETC2/EAC texture compression

A mandatory supported feature, compressed textures have obvious transmission time savings. 

```javascript
gl.compressedTexImage2D(
    gl.TEXTURE_2D, 
    0, 
    gl.COMPRESSED_RGBA8_ETC2_EAC, 
    IMAGE_SIZE.width, 
    IMAGE_SIZE.height, 
    0, 
    pixels
);
```

* Integer textures

* Non-Power-of-Two Texture
    * texturing video
    * 2D Sprite

* Floating point textures
    - half-float: High dynamic range imaging
    - full-float: Variance shadow maps soft shadow

    - a feature coming together with **floating point texture** is **floating point renderbuffer** (also with multisample support).



* Seamless cube map

Cube map is already available in WebGL 1. What's new in WebGL 2 is that the cube map is 
seamless (and is always seamless, unlike in OpenGL where you can set it). With this feature 
we are free from using hacks to get rid of the artifacts near the boarders.


* A set of additional texture formats

```javascript
textureFormats[TextureTypes.RGB] = {
    internalFormat: gl.RGB,
    format: gl.RGB,
    type: gl.UNSIGNED_BYTE
};

textureFormats[TextureTypes.RGB8] = {
    internalFormat: gl.RGB8,
    format: gl.RGB,
    type: gl.UNSIGNED_BYTE
};

textureFormats[TextureTypes.RGB16F] = {
    internalFormat: gl.RGB16F,
    format: gl.RGB,
    type: gl.HALF_FLOAT
};

textureFormats[TextureTypes.RGBA32F] = {
    internalFormat: gl.RGBA32F,
    format: gl.RGBA,
    type: gl.FLOAT
};

textureFormats[TextureTypes.R16F] = {
    internalFormat: gl.R16F,
    format: gl.RED,
    type: gl.HALF_FLOAT
};

textureFormats[TextureTypes.RG16F] = {
    internalFormat: gl.RG16F,
    format: gl.RG,
    type: gl.HALF_FLOAT
};

textureFormats[TextureTypes.RGBA] = {
    internalFormat: gl.RGBA,
    format: gl.RGBA,
    type: gl.UNSIGNED_BYTE
};

textureFormats[TextureTypes.RGB8UI] = {
    internalFormat: gl.RGB8UI,
    format: gl.RGB_INTEGER,
    type: gl.UNSIGNED_BYTE
};

textureFormats[TextureTypes.RGBA8UI] = {
    internalFormat: gl.RGBA8UI,
    format: gl.RGBA_INTEGER,
    type: gl.UNSIGNED_BYTE
};
```



## New GLSL 3.00 ES Shader

And here comes our new shader: GLSL 3.00 ES! This new version brings in a bunch of new features 
that are not in GLSL 1.00. But the grammar changed at some point, so it can be quite painful converting over 
at the start. 

Note that a shader in GLSL 1.00 is still fully supported in a WebGL 2 context. It's only the GLSL 3.00 ES grammar that 
doesn't have backwards compatibility with GLSL 1.00. 
Only when a `#version 300 es` tag is added at the top of the shaders will the GLSL 3.00 ES version turned on. 

We will quickly list here a bunch of new features and new built-in functions in GLSL 3.00 ES. 

* Layout qualifiers

Vertex shader inputs can now be declared with layout qualifiers to explicitly bind the location 
in the shader source without requiring making `gl.getAttribLocation` calls, like this: 

```GLSL
#version 300 es
#define POSITION_LOCATION 0
#define TEXCOORD_LOCATION 4
// ...
layout(location = POSITION_LOCATION) in vec2 position;
layout(location = TEXCOORD_LOCATION) in vec2 texcoord;
```

```javascript
var vertexPosLocation = 0; // set with GLSL layout qualifier
gl.enableVertexAttribArray(vertexPosLocation);
gl.bindBuffer(gl.ARRAY_BUFFER, vertexPosBuffer);
gl.vertexAttribPointer(vertexPosLocation, 2, gl.FLOAT, false, 0, 0);
gl.bindBuffer(gl.ARRAY_BUFFER, null);

var vertexTexLocation = 4; // set with GLSL layout qualifier
gl.enableVertexAttribArray(vertexTexLocation);
gl.bindBuffer(gl.ARRAY_BUFFER, vertexTexBuffer);
gl.vertexAttribPointer(vertexTexLocation, 2, gl.FLOAT, false, 0, 0);
gl.bindBuffer(gl.ARRAY_BUFFER, null);
```

The same applies to fragment shader outputs. Layout qualifiers can also be used to control the 
memory layout for uniform blocks. 

* Non-square matrix

Quite straightforward. One use case is replace a 4x4 affine matrix where the last row is (0, 0, 0, 1) with 
a 4x3 matrix. 

* Full integer support

Built-in functions can now take integer as input variable. 

* Flat/smooth interpolators

We can now explicitly declare `flat` interpolators to have flat shading. 

* Centroid sampling

This is used to avoid rendering artifacts when multisampling. Read [this article](https://www.opengl.org/pipeline/article/vol003_6/) 
for more details. Here is a [WebGL 2 Sample of centroid sampling](http://webglsamples.org/WebGL2Samples/#glsl_centroid).

* New built-in functions

Some very handy functions such as `textureOffset`, `texelFetch`, `dFdx`, `textureGrad`, `textureLOD`, etc. 
You can always find the complete lists in [GLSL 3.00 ES Spec](https://www.khronos.org/registry/gles/specs/3.0/GLSL_ES_Specification_3.00.4.pdf) 

* `gl_InstanceID` and `gl_VertexID`

These allow identification of instances and vertices within the shader.


* Other function name changes

For example, `texture2D(sampler2D sampler, vec2 coord)`, `textureCube(samplerCube sampler, vec3 coord)`, 
(and `texture3D` if there is any in our old version) are now replaced with a set of 
overrided functions of `texture`

```GLSL
gvec4 texture (gsampler2D sampler, vec2 P [, float bias] )
gvec4 texture (gsampler3D sampler, vec3 P [, float bias] )
gvec4 texture (gsamplerCube sampler, vec3 P [, float bias] )
```



# Credits

* Brandon Jones http://blog.tojicode.com/2013/09/whats-coming-in-webgl-20.html 
* Hongwei Li https://zhuanlan.zhihu.com/p/19957067?refer=webgl 
* WebGL 2 Spec https://www.khronos.org/registry/webgl/specs/latest/2.0/ 
* OpenGL ES 3 Spec https://www.khronos.org/registry/gles/specs/3.0/es_spec_3.0.0.pdf
* OpenGL ES 3 Programming Guide http://1.droppdf.com/files/v4voM/addison-wesley-opengl-es-3-0-programming-guide-2nd-2014.pdf
* http://gamedev.stackexchange.com/questions/9668/what-are-3d-textures
* http://www.gamedev.net/topic/655969-speed-gluniform-vs-uniform-buffer-objects/ 
* Sijie Tian https://hacks.mozilla.org/2014/01/webgl-deferred-shading/
* Dong Dong https://www.zhihu.com/question/49327688/answer/115691345?from=profile_answer_card 
* Cesium https://github.com/AnalyticalGraphicsInc/cesium 
* WebGL Stats http://webglstats.com/ 
* WebGL 2 for Siggraph Asia 2015 https://docs.google.com/presentation/d/1Orx0GB0cQcYhHkYsaEcoo5js3c5-pv7ahPniIRIzzfg/edit#slide=id.gd1fc5cab2_0_8
