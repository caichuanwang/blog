---
title: css-filter-滤镜
date: {{ date }}
tags: 
    - vue
    - react
categories: 
    - Web 前端
description: 这是一篇关于 css 中滤镜(filter)属性的介绍文章
---

# 滤镜



[CSS](https://developer.mozilla.org/zh-CN/docs/Web/CSS) 属性 **`filter`** 将模糊或颜色偏移等图形效果应用于元素。滤镜通常用于调整图像、背景和边框的渲染。

![9a78c5f36ff94ef297cb94bf7c4a76ef~tplv-k3u1fbpfcp-zoom-in-crop-mark_1512_0_0_0](http://gicgo-images.oss-cn-shanghai.aliyuncs.com/img/9a78c5f36ff94ef297cb94bf7c4a76ef~tplv-k3u1fbpfcp-zoom-in-crop-mark_1512_0_0_0.png)

### css滤镜

#### blur()

用于高斯模糊，内置参数从0向上，数值越大越模糊

#### drop-shadow()

参数（\<offset-x\>\<offset-y\>）必填项目，这是设置阴影偏移量的两个length值

参数（\<blur-radius\>）值越大，越模糊，所以阴影可以变得更大或更淡。不允许负值。

参数（\<color\>）

#### grayscale()灰度

从0到100%，图片线性呈现灰度

#### opacity

透明度，参数从0-100%，线性呈现出不透明

### svg滤镜

#### 滤镜原语

一般以`fe`开头，滤镜通过 [` <defs> ` ](https://developer.mozilla.org/zh-CN/docs/Web/SVG/Element/filter) 元素进行定义，并且置于 `<defs>` 区块中。在 `filter` 标签中提供一系列*图元*（*primitives*），以及在前一个基本变换操作上建立的另一个操作（比如添加模糊后又添加明亮效果）。如果要应用所创建的滤镜效果，只需要为 SVG 图形元素设置 `filter` 属性即可。

```css
<svg width="250" viewBox="0 0 200 85"
     xmlns="http://www.w3.org/2000/svg" version="1.1">
  <defs>
    <!-- Filter declaration开始一个filter -->
    <filter id="MyFilter" filterUnits="userSpaceOnUse" #这里定义这个id，很重要
            x="0" y="0"
            width="200" height="120">

      <!-- offsetBlur -->  #高斯模糊
      <feGaussianBlur in="SourceAlpha" stdDeviation="4" result="blur"/> #模糊度为4
      <feOffset in="blur" dx="4" dy="4" result="offsetBlur"/>   #向右偏移4，向下偏移4

      <!-- litPaint光照效果 -->
      <feSpecularLighting in="blur" surfaceScale="5" specularConstant=".75"
                          specularExponent="20" lighting-color="#bbbbbb"
                          result="specOut">
        <fePointLight x="-5000" y="-10000" z="20000"/> #点光源效果
      </feSpecularLighting> 
      #第二个输入源（in2）为 "SourceAlpha"，也就是原图的
			#该滤镜执行两个输入图像的智能像素组合
      <feComposite in="specOut" in2="SourceAlpha" operator="in" result="specOut"/>
      <feComposite in="SourceGraphic" in2="specOut" operator="arithmetic"
                   k1="0" k2="1" k3="1" k4="0" result="litPaint"/>

      <!-- merge offsetBlur + litPaint -->
      <feMerge>
        <feMergeNode in="offsetBlur"/>
        <feMergeNode in="litPaint"/>
      </feMerge>
    </filter>
  </defs>

  <!-- Graphic elements -->
  <g filter="url(#MyFilter)">
      <path fill="none" stroke="#D90000" stroke-width="10"
            d="M50,66 c-50,0 -50,-60 0,-60 h100 c50,0 50,60 0,60z" />
      <path fill="#D90000"
            d="M60,56 c-30,0 -30,-40 0,-40 h80 c30,0 30,40 0,40z" />
      <g fill="#FFFFFF" stroke="black" font-size="45" font-family="Verdana" >
        <text x="52" y="52">SVG</text>
      </g>
  </g>
</svg>

```

从上面的一个片段可以看出

### 滤镜根元素 - `<filter>`

`<filter>` 是 SVG 滤镜元素的根标签，存在于 `<svg>` 元素的子层级中。

在 `<filter>` 标签中，可以定义滤镜在使用元素处生效范围、坐标空间及其下方所包含的滤镜元素（原语）的坐标空间等等属性。

```
filterUnits
```

定义了滤镜生效范围的坐标系统（即滤镜根元素的 `x`、`y`、`width`、`height` 属性将如何取值）

默认值为 `objectBoundingBox`，可选值包括：`objectBoundingBox`|`userSpaceOnUse`。

`objectBoundingBox` - 相对应用滤镜的元素所占区域来设置滤镜区域，设置该值后，下方 `x`、`y`、`width`、`height` 四个值应当为分数（百分比、小数）值。

`userSpaceOnUse` - 相对 `<svg>` 标签所占区域来设置滤镜区域

```
`x`、`y`
```

表示滤镜生效范围的起始坐标。

默认值为 `-10%`。

```
width、height
```

表示滤镜生效范围的终止坐标。

默认值为 `120%`。

##### feGaussianBlur

设置高斯模糊滤镜。

##### feOffset

创造移动效果。

##### feDropshadow

设置投影滤镜 `<feDropshadow>`

开启滤镜，不设置滤镜范围 —— 此时滤镜范围将使用默认值 `x="-10%" y="-10%" width="120%" height="120%"`。



### 工作常见

#### 网页置灰

```css
filter:grayscale(100%);  #使用时注意兼容性
```

#### 远程获取svg

一开始直接使用img的src，但是发现这样没法根据`color`改变

然后使用滤镜：

```tsx
 <img
            width="16px"
            height="16px"
            key={e.navigation_name}
            style={{
              marginRight: '10px',
              verticalAlign: 'baseline',
              filter: ' drop-shadow(9999px 0 0)',
              transform: 'translate(-9999px)',
            }}
            src={e.navigation_router_prefix + e.navigation_icon}
          />
```

