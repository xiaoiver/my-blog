---
layout: post
title: "Wireframe 的实现"
cover: "/assets/img/webgl/wireframe.png"
date:   2018-10-22
category: coding
tags: WebGL
author: xiaOp
comments: true
index: 63
---

在调试或者展示模型时，有时需要展现 wireframe 网格效果，在 ThreeJS 中经常能看到，之前也没想过这是如何实现的。
最近在阅读 clayGL 的代码时，顺着里面的注释找到了这篇文章：[easy-wireframe-display-with-barycentric-coordinates](http://codeflow.org/entries/2012/aug/02/easy-wireframe-display-with-barycentric-coordinates/)。里面的实现十分巧妙，在此记录一下。

首先我们会介绍基本实现思路，然后是改进效果，最后是一些问题以及扩展实现。
另外，文中的效果图都是 GLSLCanvas 实时渲染的，因此打开控制台就能看到异步请求的 shader 源代码。

## 重心坐标

思路其实十分简单，我们想在光栅化时给每个三角形描边，那么就需要知道当前 fragment 距离三角形的三边各有多远，一旦小于边框的宽度，我们就给当前 fragment 着上边框的颜色。

所以问题的关键就是如何计算距离三角形三边的距离。上面那篇文章中使用了重心坐标，由于我们只关心当前 fragment 所在的三角形，以三个顶点构建重心坐标系，利用 fragment shader 的插值就能得到当前 fragment 对应的重心坐标。其实在光栅化过程中，会利用重心坐标作为权重来决定 fragment 的颜色（例如下图），$$C_P = \lambda_0 * C_{V0} + \lambda_1 * C_{V1} + \lambda_2 * C_{V2}$$，感兴趣可以阅读 scratchapixel 上关于光栅化具体实现的[文章](https://www.scratchapixel.com/lessons/3d-basic-rendering/rasterization-practical-implementation/rasterization-stage)。
![](/assets/img/webgl/barycentric2.png)

回到我们的问题上来，在实现中，和颜色，纹理坐标，法线等其他 vertex attribute 一样，给每个 vertex 增加一个初始重心坐标就行了：
{% prism glsl linenos %}
// vertex shader

attribute vec3 a_Barycentric;
varying vec3 v_Barycentric;
void main() {
    v_Barycentric = a_Barycentric;
}
{% endprism %}

首先给顶点传入重心坐标，我们需要保证三角形三个顶点坐标值分别是 `(1,0,0)` `(0,1,0)` 和 `(0,0,1)`。如果在绘制时使用的是 `gl.drawArrays()`，那只需要简单的按顺序依次传入三个顶点坐标，重复多次（三角形个数）就行了，我们以一个简单平面（两个三角形组成）为例：
{% prism javascript linenos %}
const vertices = new Float32Array([
    1.0, 0.0, 1.0,  1.0, 0.0, -1.0,  -1.0, 0, -1.0,
    1.0, 0.0, 1.0,  -1.0, 0, -1.0,   -1.0, 0, 1.0
]);
// [1,0,0, 0,1,0, 0,0,1, 1,0,0...]
const barycentrics = vertices.map((v, i) => {
    if (i % 9 === 0 || i % 9 === 4 || i % 9 === 8) {
        return 1.0;
    }
    return 0.0;
});
// 省略传入 vertex attributes 代码
gl.drawArrays(gl.TRIANGLES, 0, vertices.length / 3);
{% endprism %}

然后在 fragment shader 中，当重心坐标任意一个分量小于边框宽度阈值，就可以当作边框绘制。这里用到了 glsl 内置函数 `any()` 和 `lessThan()`：
{% prism glsl linenos %}
// fragment shader

varying vec3 v_Barycentric;
void main() {
    // 小于边框宽度
    if (any(lessThan(v_Barycentric, vec3(0.1)))) {
        // 边框颜色
        gl_FragColor = vec4(0.0, 0.0, 0.0, 1.0);
    }
    else {
        // 填充背景颜色
        gl_FragColor = vec4(0.5, 0.5, 0.5, 1.0);
    }
}
{% endprism %}

我们以最简单的由两个三角形组成的正方形为例，边框宽度为 1/10，效果如下：
<div class="glsl-canvas-wrapper">
    <canvas id="wireframe"
    data-vertex-url="{{ site.baseurl }}/assets/shaders/wireframe/wireframe.vert"
    data-fragment-url="{{ site.baseurl }}/assets/shaders/wireframe/wireframe.frag"
    width="500" height="500"></canvas>
</div>
<script>
const sandbox = new GlslCanvas(document.querySelector('#wireframe'));
sandbox.on('load', function () {
    try {
        const gl = sandbox.gl;
        gl.getExtension('OES_standard_derivatives');
        const program = sandbox.program;
        const positions = [
            -1.0, -1.0, 1.0, -1.0, -1.0, 1.0, -1.0, 1.0, 1.0, -1.0, 1.0, 1.0
        ];
        const barycentrics = [
            1.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 1.0,
            1.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 1.0
        ];
        if (initArrayBuffer(gl, program, 'a_Position', new Float32Array(positions), gl.FLOAT, 2)
            && initArrayBuffer(gl, program, 'a_Barycentric', new Float32Array(barycentrics), gl.FLOAT, 3)
        ) { 
            sandbox.forceRender = true;
            sandbox.render();
        }
    } catch (e) {}
});
</script>

## 改进效果

上面的实现有一个明显的问题，我们当然可以指定边框宽度为 1/100 甚至更小，但是在某些需要放大查看模型的场景中，边框也随之放大了，这通常不是我们想要的。为了保持宽度不变，我们需要根据当前屏幕空间缩放的比例调整，借助 [OES_standard_derivatives](https://www.khronos.org/registry/OpenGL/extensions/OES/OES_standard_derivatives.txt) 扩展提供的三个函数就可以做到：

* `dFdx(p)` 就是当屏幕坐标 x 改变1时，当前 `p` xyz 分量会变化多少。`dFdy(p)` 同理
* `fwidth(p)` 其实就是 `abs(dFdx(p)) + abs(dFdy(p))`，反映了平均变化

这个扩展在 WebGL2 中是默认开启的，而在 WebGL1 中使用需要手动开启：`gl.getExtension('OES_standard_derivatives');`，另外在 shader 中也要声明。详见 [MDN WebGL API](https://developer.mozilla.org/en-US/docs/Web/API/OES_standard_derivatives)。

{% prism glsl linenos %}
// fragment shader

#extension GL_OES_standard_derivatives : enable
float edgeFactor(){
    vec3 d = fwidth(v_Barycentric);
    // 边缘平滑效果
    vec3 a3 = smoothstep(vec3(0.0), d * 1.5, v_Barycentric);
    return min(min(a3.x, a3.y), a3.z);
}

void main() {
    gl_FragColor.rgb = mix(vec3(0.0), vec3(1.0), edgeFactor());
    gl_FragColor.a = 1.0;
}
{% endprism %}

效果如下：
<div class="glsl-canvas-wrapper">
    <canvas id="wireframe2"
    data-vertex-url="{{ site.baseurl }}/assets/shaders/wireframe/wireframe.vert"
    data-fragment-url="{{ site.baseurl }}/assets/shaders/wireframe/wireframe-smooth.frag"
    width="500" height="500"></canvas>
</div>
<script>
const sandbox2 = new GlslCanvas(document.querySelector('#wireframe2'));
sandbox2.on('load', function () {
try {
    const gl = sandbox2.gl;
    gl.getExtension('OES_standard_derivatives');
    const program = sandbox2.program;
    const positions = [
        -1.0, -1.0, 1.0, -1.0, -1.0, 1.0, -1.0, 1.0, 1.0, -1.0, 1.0, 1.0
    ];
    const barycentrics = [
        1.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 1.0,
        1.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 1.0
    ];
    if (initArrayBuffer(gl, program, 'a_Position', new Float32Array(positions), gl.FLOAT, 2)
        && initArrayBuffer(gl, program, 'a_Barycentric', new Float32Array(barycentrics), gl.FLOAT, 3)
    ) { 
        sandbox2.forceRender = true;
        sandbox2.render();
    }
} catch (e) {}
});
</script>

## 后续思考

基本的实现思路就是这样了，但是在我自己实现和查阅资料的过程中，遇到了一些问题，接下来让我们来看一下。

### 共享顶点

之前的例子中我们在绘制时使用了 `gl.drawArrays()`，但如果使用的是更节省 Buffer 空间的 `gl.drawElements()`，也就是共享部分顶点（例如平面仅使用 4 个而非 6 个顶点），就不能简单根据顶点顺序，得依照顶点索引分配重心坐标了。
{% prism javascript linenos %}
const vertices = new Float32Array([
    1.0, 0.0, 1.0,  1.0, 0.0, -1.0,  -1.0, 0, -1.0,   -1.0, 0, 1.0
]);
const indices = new Uint8Array([
    0, 1, 2, 0, 2, 3
]);
const barycentrics = [
    1,0,0, 0,1,0, 0,0,1, 0,1,0
];
gl.drawElements(gl.TRIANGLES, indices.length, type, 0);
{% endprism %}

但不是所有分配方式都这么简单，比如 StackOverflow 上的[这个问题](https://stackoverflow.com/questions/24839857/wireframe-shader-issue-with-barycentric-coordinates-when-using-shared-vertices)，会发现问号处无法分配。根本原因其实是在共享顶点的情况下，一旦给一个三角形分配好了重心坐标，与之共享一边的下一个三角形的剩余一个顶点坐标实际也已经确定了：
![](/assets/img/webgl/wireframe3.png)

下面的回答给出了两种解决思路：
1. 如果不要求一定要绘制正方形的对角线，只要求 4 边的话，可以放弃重心坐标
2. 按一定顺序分配，才能不出现冲突

更多细节可以进入这个问答深入了解。

### 虚线效果

既然我们能控制线的粗细，那也就有办法实现虚线效果，就是宽度在 0-1 之间进行周期性变换嘛。在这篇 [wireframe-shader-implementation](https://forum.libcinder.org/topic/wireframe-shader-implementation) 文章中就使用了 `sin()`
{% prism glsl linenos %}
float f = v_Barycentric.x;
if( v_Barycentric.x < min(v_Barycentric.y, v_Barycentric.z) )
    f = v_Barycentric.y;

const float PI = 3.14159265;
float stipple = pow( clamp( 5.0 * sin( f * 21.0 * PI ), 0.0, 1.0 ), 10.0 );
float thickness = 2.0 * stipple;
{% endprism %}

效果如下：
<div class="glsl-canvas-wrapper">
    <canvas id="wireframe3"
    data-vertex-url="{{ site.baseurl }}/assets/shaders/wireframe/wireframe.vert"
    data-fragment-url="{{ site.baseurl }}/assets/shaders/wireframe/wireframe-dash.frag"
    width="500" height="500"></canvas>
</div>
<script>
const sandbox3 = new GlslCanvas(document.querySelector('#wireframe3'));
sandbox3.on('load', function () {
try {
    const gl = sandbox3.gl;
    gl.getExtension('OES_standard_derivatives');
    const program = sandbox3.program;
    const positions = [
        -1.0, -1.0, 1.0, -1.0, -1.0, 1.0, -1.0, 1.0, 1.0, -1.0, 1.0, 1.0
    ];
    const barycentrics = [
        1.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 1.0,
        1.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 1.0
    ];
    if (initArrayBuffer(gl, program, 'a_Position', new Float32Array(positions), gl.FLOAT, 2)
        && initArrayBuffer(gl, program, 'a_Barycentric', new Float32Array(barycentrics), gl.FLOAT, 3)
    ) { 
        sandbox3.forceRender = true;
        sandbox3.render();
    }
} catch (e) {}
});
</script>

### 透明背景

之前我们给网格空隙处填充了一个背景色，更好的效果是做成透明的，这样也能看到背面的网格。
很自然想到使用 alpha 通道，网格处完全不透明：
{% prism javascript linenos %}
gl_FragColor = vec4(0.0, 0.0, 0.0, 1.0 - edgeFactor());
{% endprism %}

另外，由于复杂模型顶点众多，为了更好的观察效果，我们希望正面的网格要更清晰一些，相对的背面就可以淡化一点。
利用 fragment shader 中的输入变量 `gl_FrontFacing` 简单判断下就可以了：
{% prism javascript linenos %}
if (gl_FrontFacing) {
    gl_FragColor = vec4(0.0, 0.0, 0.0, 1.0 - edgeFactor());
}
else {
    // 淡化背面
    gl_FragColor = vec4(0.0, 0.0, 0.0, (1.0 - edgeFactor()) * 0.3);
}
{% endprism %}

看似没问题，但是实际运行效果有点奇怪，看起来正面完全没有渲染出来，这里我们使用了一个立方体：
<div class="glsl-canvas-wrapper">
    <canvas id="wireframe4" style="background: linear-gradient(#e66465, #9198e5);"
    data-vertex-url="{{ site.baseurl }}/assets/shaders/wireframe/wireframe-transparent.vert"
    data-fragment-url="{{ site.baseurl }}/assets/shaders/wireframe/wireframe-transparent.frag"
    width="500" height="500"></canvas>
</div>
<script>
const sandbox4 = new GlslCanvas(document.querySelector('#wireframe4'));
function renderPrograms () {
    const gl = this.gl;
    const W = gl.canvas.width;
    const H = gl.canvas.height;
    this.updateVariables();
    gl.viewport(0, 0, W, H);
    for (let key in this.buffers) {
        const buffer = this.buffers[key];
        this.updateUniforms(buffer.program, key);
        buffer.bundle.render(W, H, buffer.program, buffer.name);
        gl.bindFramebuffer(gl.FRAMEBUFFER, null);
    }
    this.updateUniforms(this.program, 'main');
    gl.drawArrays(gl.TRIANGLES, 0, 36);
};
sandbox4.renderPrograms = renderPrograms.bind(sandbox4);
sandbox4.on('load', function () {
try {
    const gl = sandbox4.gl;
    gl.enable(gl.BLEND);
    // gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA);
    gl.getExtension('OES_standard_derivatives');
    const program = sandbox4.program;
    let positions = [];
    let barycentrics = [];
    positions = [
      1.0, 1.0, 1.0,  -1.0, 1.0, 1.0,  -1.0,-1.0, 1.0, 1.0, 1.0, 1.0, -1.0,-1.0, 1.0, 1.0,-1.0, 1.0, // v0-v1-v2-v3 front
      1.0, 1.0, 1.0,   1.0,-1.0, 1.0,   1.0,-1.0,-1.0,  1.0, 1.0, 1.0, 1.0,-1.0,-1.0, 1.0, 1.0,-1.0, // v0-v3-v4-v5 right
      1.0, 1.0, 1.0,   1.0, 1.0,-1.0,  -1.0, 1.0,-1.0, 1.0, 1.0, 1.0, -1.0, 1.0,-1.0,-1.0, 1.0, 1.0, // v0-v5-v6-v1 up
     -1.0, 1.0, 1.0,  -1.0, 1.0,-1.0,  -1.0,-1.0,-1.0, -1.0, 1.0, 1.0, -1.0,-1.0,-1.0,-1.0,-1.0, 1.0, // v1-v6-v7-v2 left
     -1.0,-1.0,-1.0,   1.0,-1.0,-1.0,   1.0,-1.0, 1.0, -1.0,-1.0,-1.0, 1.0,-1.0, 1.0,-1.0,-1.0, 1.0, // v7-v4-v3-v2 down
      1.0,-1.0,-1.0,  -1.0,-1.0,-1.0,  -1.0, 1.0,-1.0, 1.0,-1.0,-1.0,  -1.0, 1.0,-1.0,1.0, 1.0,-1.0,  // v4-v7-v6-v5 back
    ];
    positions = positions.map(function(p) {return p/3;});
    for (let i = 0; i < positions.length / 9; i++) {
        barycentrics = barycentrics.concat([1.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 1.0]);
    }
    if (initArrayBuffer(gl, program, 'a_Position', new Float32Array(positions), gl.FLOAT, 3)
        && initArrayBuffer(gl, program, 'a_Barycentric', new Float32Array(barycentrics), gl.FLOAT, 3)
    ) { 
        const location = gl.getUniformLocation(program, 'u_MvpMatrix');
        gl.uniformMatrix4fv(location, false, new Float32Array([
            1.6590037908279933, -0.214974148922413, -0.2596793675696521, -0.25916052767440806, 0, 1.5621454821695344, -0.43279894594942014, -0.43193421279068006, -0.497701137248398, -0.7165804964080433, -0.8655978918988403, -0.8638684255813601, 3.845925372767128e-16, -2.644073693777401e-16, 1.3787915312242403, 1.5758369027902257
        ]));
        sandbox4.forceRender = true;
        sandbox4.render();
    }
} catch (e) {}
});
</script>

原因其实很简单，WebGL 默认没有开启 alpha blending，由于我们指定的 6 个面绘制顺序是前、右、上、左、下、后，因此最后绘制的左、下、后三面完全覆盖了先绘制的三面。
因此我们需要手动开启 alpha blending，并指定混合函数 [blendFunc](https://developer.mozilla.org/en-US/docs/Web/API/WebGLRenderingContext/blendFunc)。
其中两个参数分别为混合因子 `sfactor` 和 `dfactor`，最终会被应用到混合计算公式中：`color(RGBA) = (sourceColor * sfactor) + (destinationColor * dfactor)`：
{% prism javascript linenos %}
gl.enable(gl.BLEND);
gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA);
{% endprism %}

最终效果如下：
<div class="glsl-canvas-wrapper">
    <canvas id="wireframe5" style="background: linear-gradient(#e66465, #9198e5);"
    data-vertex-url="{{ site.baseurl }}/assets/shaders/wireframe/wireframe-transparent.vert"
    data-fragment-url="{{ site.baseurl }}/assets/shaders/wireframe/wireframe-transparent.frag"
    width="500" height="500"></canvas>
</div>
<script>
const sandbox5 = new GlslCanvas(document.querySelector('#wireframe5'));
function renderPrograms () {
    const gl = this.gl;
    const W = gl.canvas.width;
    const H = gl.canvas.height;
    this.updateVariables();
    gl.viewport(0, 0, W, H);
    for (let key in this.buffers) {
        const buffer = this.buffers[key];
        this.updateUniforms(buffer.program, key);
        buffer.bundle.render(W, H, buffer.program, buffer.name);
        gl.bindFramebuffer(gl.FRAMEBUFFER, null);
    }
    this.updateUniforms(this.program, 'main');
    gl.drawArrays(gl.TRIANGLES, 0, 36);
};
sandbox5.renderPrograms = renderPrograms.bind(sandbox5);
sandbox5.on('load', function () {
try {
    const gl = sandbox5.gl;
    gl.enable(gl.BLEND);
    gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA);
    gl.getExtension('OES_standard_derivatives');
    const program = sandbox5.program;
    let positions = [];
    let barycentrics = [];
    positions = [
      1.0, 1.0, 1.0,  -1.0, 1.0, 1.0,  -1.0,-1.0, 1.0, 1.0, 1.0, 1.0, -1.0,-1.0, 1.0, 1.0,-1.0, 1.0, // v0-v1-v2-v3 front
      1.0, 1.0, 1.0,   1.0,-1.0, 1.0,   1.0,-1.0,-1.0,  1.0, 1.0, 1.0, 1.0,-1.0,-1.0, 1.0, 1.0,-1.0, // v0-v3-v4-v5 right
      1.0, 1.0, 1.0,   1.0, 1.0,-1.0,  -1.0, 1.0,-1.0, 1.0, 1.0, 1.0, -1.0, 1.0,-1.0,-1.0, 1.0, 1.0, // v0-v5-v6-v1 up
     -1.0, 1.0, 1.0,  -1.0, 1.0,-1.0,  -1.0,-1.0,-1.0, -1.0, 1.0, 1.0, -1.0,-1.0,-1.0,-1.0,-1.0, 1.0, // v1-v6-v7-v2 left
     -1.0,-1.0,-1.0,   1.0,-1.0,-1.0,   1.0,-1.0, 1.0, -1.0,-1.0,-1.0, 1.0,-1.0, 1.0,-1.0,-1.0, 1.0, // v7-v4-v3-v2 down
      1.0,-1.0,-1.0,  -1.0,-1.0,-1.0,  -1.0, 1.0,-1.0, 1.0,-1.0,-1.0,  -1.0, 1.0,-1.0,1.0, 1.0,-1.0  // v4-v7-v6-v5 back
    ];
    positions = positions.map(function(p) {return p/3;});
    for (let i = 0; i < positions.length / 9; i++) {
        barycentrics = barycentrics.concat([1.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 1.0]);
    }
    if (initArrayBuffer(gl, program, 'a_Position', new Float32Array(positions), gl.FLOAT, 3)
        && initArrayBuffer(gl, program, 'a_Barycentric', new Float32Array(barycentrics), gl.FLOAT, 3)
    ) { 
        const location = gl.getUniformLocation(program, 'u_MvpMatrix');
        gl.uniformMatrix4fv(location, false, new Float32Array([
            1.6590037908279933, -0.214974148922413, -0.2596793675696521, -0.25916052767440806, 0, 1.5621454821695344, -0.43279894594942014, -0.43193421279068006, -0.497701137248398, -0.7165804964080433, -0.8655978918988403, -0.8638684255813601, 3.845925372767128e-16, -2.644073693777401e-16, 1.3787915312242403, 1.5758369027902257
        ]));
        sandbox5.forceRender = true;
        sandbox5.render();
    }
} catch (e) {}
});
</script>

## 总结

本文介绍了使用重心坐标在 single-pass 中完成 wireframe 的绘制。
另外，在查阅资料的时候我发现一篇 [single-pass-wireframe-rendering](http://strattonbrazil.blogspot.com/2011/09/single-pass-wireframe-rendering_10.html) 其中使用到了 Geometry Shader，但是在 WebGL 中[并不支持 Geometry shader](https://news.ycombinator.com/item?id=13893412)。

## 参考资料

* [easy-wireframe-display-with-barycentric-coordinates](http://codeflow.org/entries/2012/aug/02/easy-wireframe-display-with-barycentric-coordinates/)
* [rasterization-practical-implementation](https://www.scratchapixel.com/lessons/3d-basic-rendering/rasterization-practical-implementation/rasterization-stage)
* [wireframe-shader-issue-with-barycentric-coordinates-when-using-shared-vertices](https://stackoverflow.com/questions/24839857/wireframe-shader-issue-with-barycentric-coordinates-when-using-shared-vertices)
* [wireframe-shader-implementation](https://forum.libcinder.org/topic/wireframe-shader-implementation)
* [single-pass-wireframe-rendering](http://strattonbrazil.blogspot.com/2011/09/single-pass-wireframe-rendering_10.html)
* [WebGL 单通道wireframe渲染](https://zhuanlan.zhihu.com/p/43139658)