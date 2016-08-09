[TODO: Opening] Bla bla bla

Some of the new features are already available in WebGL 1 as extensions, 
but will be part of the core spec in WebGL 2, which means support is guaranteed. 
In this first blog, we are going to focus on these promoted features, together with
potential compatiability issue they may cause. 
Let’s take a look at how the code is going to change. 

## Multiple Render Targets - Deferred Rendering

A big one. 

### WebGL 1

For MRT, we used `webgl_draw_buffer` extension as a work round to write g-buffers in a single pass. 
Though it is widely supported, the extension-style code doesn’t make us feel good:

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

With MRT, our codes become neat and clean in WebGL 2. 

```javascript
gl.framebufferTexture2D(gl.DRAW_FRAMEBUFFER, gl.COLOR_ATTACHMENT0, gl.TEXTURE_2D, tex[0], 0);
gl.framebufferTexture2D(gl.DRAW_FRAMEBUFFER, gl.COLOR_ATTACHMENT1, gl.TEXTURE_2D, tex[1], 0);
gl.framebufferTexture2D(gl.DRAW_FRAMEBUFFER, gl.COLOR_ATTACHMENT2, gl.TEXTURE_2D, tex[2], 0);
```

Instead of mapping color attachments to draw buffer, 
we directly use multiple `out` in the fragment shader. 
This actually benefits from the new GLSL 300 es, which will be introduced in details later. 
But it is straight forward enough. 

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

Additionally, since **Texture 2D Array** is available now, we can also choose to 
render to different layer of a texture 2d array instead of separate 2d textures. 

```javascript
gl.framebufferTextureLayer(gl.DRAW_FRAMEBUFFER, gl.COLOR_ATTACHMENT0, texture, 0, 0);
gl.framebufferTextureLayer(gl.DRAW_FRAMEBUFFER, gl.COLOR_ATTACHMENT1, texture, 0, 1);
gl.framebufferTextureLayer(gl.DRAW_FRAMEBUFFER, gl.COLOR_ATTACHMENT2, texture, 0, 2);
```










# Credits

* Brandon Jones http://blog.tojicode.com/2013/09/whats-coming-in-webgl-20.html 
* Hongwei Li https://zhuanlan.zhihu.com/p/19957067?refer=webgl 
* WebGL 2 Spec https://www.khronos.org/registry/webgl/specs/latest/2.0/ 
* OpenGL ES 3 Spec https://www.khronos.org/registry/gles/specs/3.0/es_spec_3.0.0.pdf
* http://gamedev.stackexchange.com/questions/9668/what-are-3d-textures
* http://www.gamedev.net/topic/655969-speed-gluniform-vs-uniform-buffer-objects/ 
* Sijie Tian https://hacks.mozilla.org/2014/01/webgl-deferred-shading/
* Dong Dong https://www.zhihu.com/question/49327688/answer/115691345?from=profile_answer_card 