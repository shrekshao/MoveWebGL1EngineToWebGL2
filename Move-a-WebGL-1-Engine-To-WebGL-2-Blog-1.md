WebGL 2 is now coming! Google Chrome just announced 100% of the WebGL 2 
conformance Suite is passing* (on the first configurations)
at SIGGRAPH 2016. 

Now I have an engine works well in WebGL 1, how am I going to improve that 
to WebGL 2? Things I may wonder:  
* What has to be changed?
* What can be done in a better way?
* What new features can I play with?


# How do I get WebGL 2 working

## Get a WebGL 2 Implementation (Browser)

You may have seen this for many times, let's just hit the point
* Just do it: [Getting a WebGL Implementation](https://www.khronos.org/webgl/wiki/Getting_a_WebGL_Implementation)
* Check if your current browser support it: [WebGL Report](https://www.khronos.org/webgl/wiki/Getting_a_WebGL_Implementation)

## Get a WebGL 2 Context

Programmers always try to support as many browsers as possible. So do I. 
On top the WebGL 1 version of getContext, we will try to access WebGL 2 first. 
If failed, then just back to WebGL 1 routine. 
We use Cesium as an example: 

```javascript
var defaultToWebgl2 = false;
var webgl2Supported = (typeof WebGL2RenderingContext !== 'undefined');
var webgl2 = false;
var glContext;

if (defaultToWebgl2 && webgl2Supported) {
    glContext = canvas.getContext('webgl2', webglOptions) || canvas.getContext('experimental-webgl2', webglOptions) || undefined;
    if (defined(glContext)) {
        webgl2 = true;
    }
}
if (!defined(glContext)) {
    glContext = canvas.getContext('webgl', webglOptions) || canvas.getContext('experimental-webgl', webglOptions) || undefined;
}
if (!defined(glContext)) {
    throw new RuntimeError('The browser supports WebGL, but initialization failed.');
}
```




# Promoted Features

Some of the new features are already available in WebGL 1 as extensions, 
but will be part of the core spec in WebGL 2, which means support is guaranteed. 
In this first blog entry we are going to focus on these promoted features, together with
potential compatibility issues they may cause. 
Let’s take a look at how the code changes. 

## Multiple Render Targets - Deferred Rendering

A big one. 

### WebGL 1

For MRT, we used `WEBGL_draw_buffers` extension as a work-around to write g-buffers in a single pass. 
Though it is widely supported (50%+ actually according to webgl stats), the extension-style code doesn’t make us feel good:

```javascript
var ext = gl.getExtension('WEBGL_draw_buffers');
if (!ext) {
  // ...
}
```

We then bind multiple textures, tx[] in the example below, 
to different framebuffer color attachments.

```javascript
var fb = gl.createFramebuffer();
gl.bindFramebuffer(gl.FRAMEBUFFER, fb);
gl.framebufferTexture2D(gl.FRAMEBUFFER, ext.COLOR_ATTACHMENT0_WEBGL, gl.TEXTURE_2D, tx[0], 0);
gl.framebufferTexture2D(gl.FRAMEBUFFER, ext.COLOR_ATTACHMENT1_WEBGL, gl.TEXTURE_2D, tx[1], 0);
gl.framebufferTexture2D(gl.FRAMEBUFFER, ext.COLOR_ATTACHMENT2_WEBGL, gl.TEXTURE_2D, tx[2], 0);
gl.framebufferTexture2D(gl.FRAMEBUFFER, ext.COLOR_ATTACHMENT3_WEBGL, gl.TEXTURE_2D, tx[3], 0);
```

Next we map the color attachments to draw buffer 
slots that the fragment shader will write to using `gl_FragData`.

```javascript
ext.drawBuffersWEBGL([
  ext.COLOR_ATTACHMENT0_WEBGL, // gl_FragData[0]
  ext.COLOR_ATTACHMENT1_WEBGL, // gl_FragData[1]
  ext.COLOR_ATTACHMENT2_WEBGL, // gl_FragData[2]
  ext.COLOR_ATTACHMENT3_WEBGL  // gl_FragData[3]
]);
```

Also, an extra flag is needed in the shader:

```glsl
#extension GL_EXT_draw_buffers : require
precision highp float;
// ...
void main() {
    gl_FragData[0] = vec4( v_position.xyz, 1.0 );
    gl_FragData[1] = vec4( v_normal.xyz, 1.0 );
    gl_FragData[2] = texture2D( u_colmap, v_uv );
    gl_FragData[3] = texture2D( u_normap, v_uv );
}
```



### WebGL 2

For MRT our code becomes neat and clean in WebGL 2. 

```javascript
gl.framebufferTexture2D(gl.DRAW_FRAMEBUFFER, gl.COLOR_ATTACHMENT0, gl.TEXTURE_2D, tex[0], 0);
gl.framebufferTexture2D(gl.DRAW_FRAMEBUFFER, gl.COLOR_ATTACHMENT1, gl.TEXTURE_2D, tex[1], 0);
gl.framebufferTexture2D(gl.DRAW_FRAMEBUFFER, gl.COLOR_ATTACHMENT2, gl.TEXTURE_2D, tex[2], 0);
```

Instead of mapping color attachments to the draw buffer, 
we directly use multiple `out` variables in the fragment shader. 
This code actually benefits from the new GLSL 3.00 ES, which we will discuss later in detail. 
However, using `out` itself is straightforward. 

```glsl
#version 300 es
precision highp float;
layout(location = 0) out vec4 gbuf_position;
layout(location = 1) out vec4 gbuf_normal;
layout(location = 2) out vec4 gbuf_colmap;
layout(location = 3) out vec4 gbuf_normap;
//...
void main()
{
    gbuf_position = vec4( v_position.xyz, 1.0 );
    gbuf_normal = vec4( v_normal.xyz, 1.0 );
    gbuf_colmap = texture2D( u_colmap, v_uv );
    gbuf_normap = texture2D( u_normap, v_uv );
}
```

Additionally, since **Texture 2D Array** is now available, we can choose to 
render to different layers of an array of texture 2d's instead of separate 2d textures. 

```javascript
gl.framebufferTextureLayer(gl.DRAW_FRAMEBUFFER, gl.COLOR_ATTACHMENT0, texture, 0, 0);
gl.framebufferTextureLayer(gl.DRAW_FRAMEBUFFER, gl.COLOR_ATTACHMENT1, texture, 0, 1);
gl.framebufferTextureLayer(gl.DRAW_FRAMEBUFFER, gl.COLOR_ATTACHMENT2, texture, 0, 2);
```




## Instancing

Exposed through the `ANGLE_instanced_arrays` extension at WebGL 1 age (90%+ support). 
Now we can simply use `drawArraysInstanced` or 
`drawArraysInstanced` for the draw calls. 

Instancing is a great performance booster for certain types of geometry, especially those in 
a great number but without too many vertices, grass and fur, for example. 




## Vertex Array Object 

TODO: more to do with engine design optimization



## Fragment Depth

Exposed through the `EXT_frag_depth` extension in WebGL 1 (66%). 

TODO: no practice yet

>Lets you manipulate the depth of a fragment from the fragment shader. 
This can be expensive because it forces the GPU to bypass a lot of it's normal fragment discard behavior, but can also allow for some interesting effects that would be difficult to accomplish without having incredibly high poly geometry.




# Other compatibility issues





# Credits

* Brandon Jones http://blog.tojicode.com/2013/09/whats-coming-in-webgl-20.html 
* Hongwei Li https://zhuanlan.zhihu.com/p/19957067?refer=webgl 
* WebGL 2 Spec https://www.khronos.org/registry/webgl/specs/latest/2.0/ 
* OpenGL ES 3 Spec https://www.khronos.org/registry/gles/specs/3.0/es_spec_3.0.0.pdf
* http://gamedev.stackexchange.com/questions/9668/what-are-3d-textures
* http://www.gamedev.net/topic/655969-speed-gluniform-vs-uniform-buffer-objects/ 
* Sijie Tian https://hacks.mozilla.org/2014/01/webgl-deferred-shading/
* Dong Dong https://www.zhihu.com/question/49327688/answer/115691345?from=profile_answer_card 
* Cesium https://github.com/AnalyticalGraphicsInc/cesium 
* WebGL Stats http://webglstats.com/ 
* WebGL 2 for Siggraph Asia 2015 https://docs.google.com/presentation/d/1Orx0GB0cQcYhHkYsaEcoo5js3c5-pv7ahPniIRIzzfg/edit#slide=id.gd1fc5cab2_0_8