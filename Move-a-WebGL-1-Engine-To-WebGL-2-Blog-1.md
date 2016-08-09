[TODO: Opening] Bla bla bla

Some of the new features are already available in WebGL 1 as extensions, 
but will be part of the core spec in WebGL 2, which means support is guaranteed. 
In this first blog, we are going to focus on these promoted features, together with
potential compatiability issue they may cause. 
Let’s take a look at how the code is going to change. 

## Multiple Render Targets

A big one for engines using modern deferred rendering techniques. 

We used `webgl_draw_buffer` extension as a work round in WebGL 1. 
Though it is widely supported, the extension-style code doesn’t make us feel good:

```javascript
var ext = gl.getExtension('WEBGL_draw_buffers');
if (!ext) {
  // ...
}
``` 

Also, an extra flag is needed in the shader:

```glsl
#extension GL_EXT_draw_buffers : require
precision mediump float;
void main() {
    gl_FragData[0] = vec4(1.0, 0.0, 0.0, 1.0);
    gl_FragData[1] = vec4(0.0, 1.0, 0.0, 1.0);
    gl_FragData[2] = vec4(0.0, 0.0, 1.0, 1.0);
    gl_FragData[3] = vec4(1.0, 1.0, 1.0, 1.0);
}
```






# Credits

* Brandon Jones http://blog.tojicode.com/2013/09/whats-coming-in-webgl-20.html 
* Hongwei Li https://zhuanlan.zhihu.com/p/19957067?refer=webgl 
* WebGL 2 Spec https://www.khronos.org/registry/webgl/specs/latest/2.0/ 
* OpenGL ES 3 Spec https://www.khronos.org/registry/gles/specs/3.0/es_spec_3.0.0.pdf
* http://gamedev.stackexchange.com/questions/9668/what-are-3d-textures
* http://www.gamedev.net/topic/655969-speed-gluniform-vs-uniform-buffer-objects/ 
* Dong Dong https://www.zhihu.com/question/49327688/answer/115691345?from=profile_answer_card 