---
layout: post
title: "Lensflare"
subtitle: "模拟相机效果"
cover: "https://intranetproxy.alipay.com/skylark/lark/0/2019/png/158945/1549443420530-b4e0077f-c0a0-417a-862f-8f75b758d375.png#align=left&display=inline&height=798&linkTarget=_blank&name=image.png&originHeight=1596&originWidth=2778&size=3578605&width=1389"
date:   2019-02-06
category: coding
tags: WebGL
author: xiaOp
comments: true
index: 77
---


[webgl_lensflares](https://threejs.org/examples/#webgl_lensflares)<br />[pseudo-lens-flare](http://john-chapman-graphics.blogspot.com/2013/02/pseudo-lens-flare.html)

镜头光晕作为一种摄影现象，能够增加某些场景下的真实感：<br />![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/158945/1549376596266-49ca644a-6c76-4a89-afdc-2f66761a3555.png#align=left&display=inline&height=446&linkTarget=_blank&name=image.png&originHeight=683&originWidth=777&size=469393&width=507)

来自 Unreal：<br />[https://docs.unrealengine.com/en-us/Engine/Rendering/PostProcessEffects/LensFlare](https://docs.unrealengine.com/en-us/Engine/Rendering/PostProcessEffects/LensFlare)<br />![](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/158945/1549458277919-6b804719-8dcc-4d45-9e57-bc5f25e9f45e.png#align=left&display=inline&height=332&linkTarget=_blank&originHeight=523&originWidth=1174&size=0&width=746)

下面介绍一种并非基于物理的生成算法，可以通过后处理实现。

## 算法

步骤如下：
1. Downsample/threshold.
1. Generate lens flare features.
1. Blur.
1. Upscale/blend with original image.

## WebGL 实现

这部分在 clay.gl 中归入了 HDR 渲染管线中。

### 降采样
由于我们需要找到场景中亮度最高的区域，所以进行降采样将显著提升后续后处理的性能：<br />![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/158945/1549378337874-a1827404-e6b3-43ce-9531-a37fea191463.png#align=left&display=inline&height=171&linkTarget=_blank&name=image.png&originHeight=217&originWidth=638&size=90471&width=504)

来自 clay.gl 的 compositor 中的第一个 pass，在 clay.gl 中创建 compositor 可以传入一个 JSON。<br />这里需要简单解释几点：
* source 就是输入的纹理
* outputs.color.parameters 中指定了输出尺寸为输入的 1/4，降采样
* defines 表示并不需要开启 RGBM
{% prism json linenos %}
// example/assets/fx/composite.json
{
  "name": "source_half",
  "shader": "#source(clay.compositor.downsample)",
  "inputs": {
    "texture": "source"
  },
  "outputs": {
    "color": {
      "parameters": {
        "width": "expr(width * dpr / 2)",
        "height": "expr(height * dpr / 2)"
      }
    }
  },
  "parameters" : {
    "textureSize": "expr( [width * dpr, height * dpr] )"
  },
  "defines": {
    "RGBM": null
  }
},
{% endprism %}

虽然并不需要应用 RGBM 编码，但仍需要引入 RGBM 模块，因为为了命名统一，decode/encodeHDR 方法包含在这个模块中，尽管都只是简单返回输入的颜色值。在这一 pass 中得到了每个 fragment 颜色的平均值，采样周围四个邻居：
{% prism glsl linenos %}
@export clay.compositor.downsample

uniform sampler2D texture;
uniform vec2 textureSize : [512, 512];

varying vec2 v_Texcoord;

// 引入 rgbm 模块但是并不会实际应用 RGBM 编码
@import clay.util.rgbm
@import clay.util.clamp_sample

void main()
{
		// 上下左右偏移量
    vec4 d = vec4(-1.0, -1.0, 1.0, 1.0) / textureSize.xyxy;
    vec4 color = decodeHDR(clampSample(texture, v_Texcoord + d.xy));
    color += decodeHDR(clampSample(texture, v_Texcoord + d.zy));
    color += decodeHDR(clampSample(texture, v_Texcoord + d.xw));
    color += decodeHDR(clampSample(texture, v_Texcoord + d.zw));
    color *= 0.25;

    gl_FragColor = encodeHDR(color);
}

@end
{% endprism %}

此时降采样后的效果如图：clay.gl 的 pisa.hdr 场景：<br />![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/158945/1549382805155-13d18293-e519-4ca0-91dc-e47b5b6267c0.png#align=left&display=inline&height=178&linkTarget=_blank&name=image.png&originHeight=1596&originWidth=2778&size=2625214&width=310)![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/158945/1549382771602-121572ad-43e4-4a40-8129-701861c5b13f.png#align=left&display=inline&height=179&linkTarget=_blank&name=image.png&originHeight=1596&originWidth=2778&size=1843024&width=311)

然后进入提取亮度 pass：
{% prism json linenos %}
{
  "name" : "bright",
  "shader" : "#source(clay.compositor.bright)",
  "inputs" : {
    "texture" : "source_half"
  },
  "defines": {
    "RGBM": null,
    "ANTI_FLICKER": null
  }
},
{% endprism %}

我们需要找到场景中亮度较高的区域，其余区域清空。这个过程可以重复多次，例如 clay.gl 中后续还会进行一次 bright2 pass：
{% prism glsl linenos %}
@export clay.compositor.bright

uniform sampler2D texture;

uniform float threshold : 1; // 最小亮度阈值
uniform float scale : 1.0; // 放大亮度因子

uniform vec2 textureSize: [512, 512];
varying vec2 v_Texcoord;

const vec3 lumWeight = vec3(0.2125, 0.7154, 0.0721);

@import clay.util.rgbm

void main()
{
    vec4 texel = decodeHDR(texture2D(texture, v_Texcoord));
    // 从 rgb 提取亮度
    float lum = dot(texel.rgb , lumWeight);
    vec4 color;
    if (lum > threshold && texel.a > 0.0)
    {
        color = vec4(texel.rgb * scale, texel.a * scale);
    }
    else
    {
        color = vec4(0.0);
    }

    gl_FragColor = encodeHDR(color);
}
@end
{% endprism %}

此时效果如图，clay.gl 原始例子中 threshold 设置过高导致完全漆黑，这里使用默认值 1：<br />![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/158945/1549382935413-2c9f3f22-b6eb-4224-a514-b004135a05c7.png#align=left&display=inline&height=212&linkTarget=_blank&name=image.png&originHeight=1596&originWidth=2778&size=219061&width=369)

接下来可以进行多次降采样 pass：bright_downsample_4、bright_downsample_8、bright_downsample_32。<br />![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/158945/1549383232919-d8ab0f90-4dc2-494b-99e5-df6eb7bd6e88.png#align=left&display=inline&height=213&linkTarget=_blank&name=image.png&originHeight=1596&originWidth=2778&size=459188&width=371)

### 生成光晕

这里传入的参数中包括 lensColor 这个纹理：
{% prism json linenos %}
{
  "name" : "lensflare",
  "shader" : "#source(clay.compositor.lensflare)",
  "inputs" : {
    "texture" : "bright2"
  },
  "parameters" : {
    "textureSize" : "expr([width * dpr / 2, height * dpr / 2])",
    "lensColor" : "#lenscolor"
  },
  "defines": {
    "RGBM": null
  }
},
{% endprism %}

#### Ghost

首先是 “Ghost”，生成方式是将上一步中高亮度区域以屏幕中心进行反转
> "Ghosts" are the repetitious blobs which mirror bright spots in the input, pivoting around the image centre. The approach I've take to generate these is to get a vector from the current pixel to the centre of the screen, then take a number of samples along this vector. 

![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/158945/1549417606598-112a11a6-8ba0-4d54-b194-1c1e089fd854.png#align=left&display=inline&height=217&linkTarget=_blank&name=image.png&originHeight=240&originWidth=404&size=95789&width=365)

按照生成方式：
* 首先需要对亮度区域的纹理坐标进行翻转后平移到中心点
* 然后生成多个 Ghost，通过 fract 实现扭曲效果
* 另外为了让屏幕中心的高亮区域得到重视，离中心越近，weight 越高
* 加上 chromatic distortion，沿采样向量偏移 rgb，会增加三次纹理的读取
* 最后从输入的 lensColor 1D 纹理中读取光晕颜色
{% prism glsl linenos %}
@export clay.compositor.lensflare

#define SAMPLE_NUMBER 8

uniform sampler2D texture;
uniform sampler2D lenscolor;

uniform vec2 textureSize : [512, 512];

uniform float dispersal : 0.3; // 色散
uniform float distortion : 1.0;

void main()
{
		// 翻转纹理坐标
    vec2 texcoord = -v_Texcoord + vec2(1.0);
    vec2 textureOffset = 1.0 / textureSize;
		// 移动到屏幕中心
    vec2 ghostVec = (vec2(0.5) - texcoord) * dispersal;
    
    vec3 distortion = vec3(-textureOffset.x * distortion, 0.0, textureOffset.x * distortion);

    vec4 result = vec4(0.0);
    for (int i = 0; i < SAMPLE_NUMBER; i++)
    {
        vec2 offset = fract(texcoord + ghostVec * float(i));
				
        // 只有中心的高亮度区域
        float weight = length(vec2(0.5) - offset) / length(vec2(0.5));
        weight = pow(1.0 - weight, 10.0);
				
        // chromatic distortion
        result += textureDistorted(offset, normalize(ghostVec), distortion) * weight;
    }
	
  	// 模拟光晕七彩颜色
    result *= texture2D(lenscolor, vec2(length(vec2(0.5) - texcoord)) / length(vec2(0.5)));
}
{% endprism %}

此时效果如下：<br />![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/158945/1549435831364-0d093af4-58b1-430d-ac33-2b02e4c566a1.png#align=left&display=inline&height=215&linkTarget=_blank&name=image.png&originHeight=1596&originWidth=2778&size=1911223&width=375)

#### Halo

限制效果在一个圆环内：
{% prism glsl linenos %}
uniform float haloWidth : 0.4;

// 光环向量
vec2 haloVec = normalize(ghostVec) * haloWidth;

float weight = length(vec2(0.5) - fract(texcoord + haloVec)) / length(vec2(0.5));
weight = pow(1.0 - weight, 10.0);
vec2 offset = fract(texcoord + haloVec);
result += textureDistorted(offset, normalize(ghostVec), distortion) * weight;
{% endprism %}

效果如下：<br />![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/158945/1549436794994-d01bed57-020e-4af2-a5db-87b49a7c1c1e.png#align=left&display=inline&height=227&linkTarget=_blank&name=image.png&originHeight=1596&originWidth=2778&size=310291&width=395)

### 模糊

如果不加模糊，会影响场景本身的效果。简单加入两个水平垂直方向的两趟高斯模糊：
{% prism json linenos %}
{
  "name" : "lensflare_blur_h",
  "shader" : "#source(clay.compositor.gaussian_blur)",
  "inputs" : {
    "texture" : "lensflare"
  },
  "outputs" : {
    "color" : {
      "parameters" : {
        "width" : "expr(width * dpr / 2)",
        "height" : "expr(height * dpr / 2)"
      }
    }
  },
  "parameters" : {
    "blurSize" : 1,
    "blurDir": 0.0,
    "textureSize" : "expr([width * dpr / 2, height * dpr / 2])"
  },
  "defines": {
    "RGBM": null
  }
},
{
  "name" : "lensflare_blur_v",
  "shader" : "#source(clay.compositor.gaussian_blur)",
  "inputs" : {
    "texture" : "lensflare_blur_h"
  },
  "outputs" : {
    "color" : {
      "parameters" : {
        "width" : "expr(width * dpr / 2)",
        "height" : "expr(height * dpr / 2)"
      }
    }
  },
  "parameters" : {
    "blurSize" : 1,
    "blurDir": 1.0,
    "textureSize" : "expr([width * dpr / 2, height * dpr / 2])"
  },
  "defines": {
    "RGBM": null
  }
},
{% endprism %}

效果如下：<br />![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/158945/1549437472056-d17398fa-ab7b-4c9b-a9e4-e20f17140552.png#align=left&display=inline&height=211&linkTarget=_blank&name=image.png&originHeight=1596&originWidth=2778&size=415056&width=368)![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/158945/1549438402740-e3717c13-7d79-497f-b40e-b2a9fb3c1cb3.png#align=left&display=inline&height=212&linkTarget=_blank&name=image.png&originHeight=1596&originWidth=2778&size=2579171&width=369)

### 混合

将上一步模糊后的 lensflare 与场景混合，暂时忽略全局辉光效果：
{% prism json linenos %}
{
  "name" : "composite",
  "shader" : "#source(clay.compositor.hdr.composite)",
  "inputs" : {
    "texture" : "source",
    "bloom" : "bloom_composite", // 暂时忽略
    "lensflare" : "lensflare_blur_v"
  },
  "outputs" : {
    "color" : {
      "parameters" : {
        "width" : "expr(width * dpr)",
        "height" : "expr(height * dpr)"
      }
    }
  },
  "defines": {
    "RGBM_DECODE": null
  }
},
{% endprism %}

下面来看两个模拟镜头真实感的附加效果。

#### 镜头灰尘效果

真实的镜头存在落灰，我们可以读取这样的纹理：<br />![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/158945/1549438105760-b70ac4a1-2ca6-4455-ad11-b114cff01ad7.png#align=left&display=inline&height=362&linkTarget=_blank&name=image.png&originHeight=724&originWidth=1024&size=638313&width=512)，

在 hdr 合成时读取纹理值：
{% prism glsl linenos %}
uniform sampler2D texture; // 愿场景
uniform sampler2D lensflare; // 模糊后的 lensflare
uniform sampler2D lensdirt; // 镜头灰尘
uniform float lensflareIntensity : 1;

texel += decodeHDR(texture2D(lensflare, v_Texcoord))
	* texture2D(lensdirt, v_Texcoord)
  * lensflareIntensity;
{% endprism %}

效果如下：<br />![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/158945/1549438774780-ef3eb286-4390-41ae-9a6e-8749c8f0f642.png#align=left&display=inline&height=798&linkTarget=_blank&name=image.png&originHeight=1596&originWidth=2778&size=3318615&width=1389)

#### DIFFRACTION STARBURST

和落灰现象一样，这种光向四周发散的效果也可以通过混合额外的纹理实现：<br />![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/158945/1549439112658-bc3c9c23-dbc2-4f28-bf29-89f371568f14.png#align=left&display=inline&height=280&linkTarget=_blank&name=image.png&originHeight=300&originWidth=800&size=157332&width=746)

这部分 clay.gl 并没有实现，但是修改也很简单：
{% prism glsl linenos %}
texel += decodeHDR(texture2D(lensflare, v_Texcoord))
        * (texture2D(lensdirt, v_Texcoord) + texture2D(lensstar, v_Texcoord))
        * lensflareIntensity;
{% endprism %}

效果如下，注意右下角的向外发散效果是固定不变的：<br />![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/158945/1549439740431-5979d07d-b3f7-477f-9aee-0a57d22a6604.png#align=left&display=inline&height=798&linkTarget=_blank&name=image.png&originHeight=1596&originWidth=2778&size=3172401&width=1389)

但是有一点和镜头灰尘不同，由于相机会发生移动，starburst 纹理的位置也需要跟随。<br />以下在 clay.gl 的 pp_lensflare 例子中进行修改，在 frame 中实时获取相机 view 矩阵：
{% prism javascript linenos %}
var camx = sceneNode.camera.viewMatrix.x;
var camz = sceneNode.camera.viewMatrix.z;
var camrot = camx.z + camz.y;

var scaleBias1 = new clay.Matrix3();
var scaleBias2 = new clay.Matrix3();
var rotateMatrix = new clay.Matrix3();
rotateMatrix.rotate(camrot);

scaleBias1.setArray([
  2.0,   0.0,  -1.0,
  0.0,   2.0,  -1.0,
  0.0,   0.0,   1.0,
]);

scaleBias2.setArray = ([
  0.5,   0.0,   0.5,
  0.0,   0.5,   0.5,
  0.0,   0.0,   1.0,
]);

rotateMatrix.multiplyLeft(scaleBias1).mul(scaleBias2);
finalCompositeNode.setParameter('lensstarMatrix', rotateMatrix.toArray());
{% endprism %}

旋转纹理坐标：
{% prism glsl linenos %}
uniform mat3 lensstarMatrix;

vec2 lensstarTexcoord = (lensstarMatrix * vec3(v_Texcoord, 1.0)).xy;
texel += decodeHDR(texture2D(lensflare, v_Texcoord))
        * (texture2D(lensdirt, v_Texcoord) + texture2D(lensstar, lensstarTexcoord))
        * lensflareIntensity;
{% endprism %}

效果如下，随着相机的移动，光圈也随之旋转：<br />![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/158945/1549443420530-b4e0077f-c0a0-417a-862f-8f75b758d375.png#align=left&display=inline&height=798&linkTarget=_blank&name=image.png&originHeight=1596&originWidth=2778&size=3578605&width=1389)
