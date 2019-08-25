title: 修改Hexo NexT主题的配色
author: gwy15
tags:
  - Hexo
  - css
categories:
  - html
date: 2019-08-25 11:55:00
---
Hexo 的 NexT 原来的主题是极简的黑白配色，一眼看上去还可以，但是看久了不免会有些视觉疲倦。如果想修改 hexo NexT 主题的配色应该怎么做呢？我在网上找到了一篇[相关的文章](https://i.zhouhuix.cn/2016/11/24/修改Hexo的Next主题/)。

这个文章中的修改方法的原理是修改`.styl`文件（Stylus语法）中的几个大的定义。按照这个方法虽然可以修改，但是会将其他部分的颜色也一起修改了，有没有更好的、更细粒度的方法呢？

<!-- more -->

### 版本信息

Hexo 版本：3.9.0

NexT 版本：7.3.0

使用的 scheme: Pisces

### 修改站名部分的颜色

站名部分就是那个大黑色方块，准确描述是 `div.site-meta`。显然其颜色经过计算之后是 `themes/next/source/css/_variables/base.styl` 文件中的 `$black-deep` 变量。但是直接修改该变量会使得下方的用户名也一起改变，因为其颜色也是定义的 `$black-deep` 变量，这不是我想要的效果。

既然使用的是 `$black-deep` 变量，我们找到其传入的地方修改不就好了吗？

在代码中查找 `$black-deep` 和 `.site-meta`，很快可以定位到相关定义，在 `themes/next/source/css/_schemes/Pieces/_brand.styl` 中，有对 `.site-meta` 的定义：
```css
.site-meta {
  background: $black-deep;
  color: white;
  padding: 20px 0;

  +tablet-mobile() {
    box-shadow: 0 0 16px rgba(0, 0, 0, .5);
  }
}
```
那么我们修改 `$black-deep` 就可以了。

这里你可以直接将 `$black-deep` 修改为你喜欢的颜色，譬如 `#fbbdd6`。但是我更建议你在 `base.styl` 文件中定义一个变量，如 `$theme-color = #fbbdd6`，再将`$black-deep` 修改为 `$theme-color`。

定义一个变量的原因后期维护会更方便。另一个理由是，如果你仅仅在 `_brand.styl` 中修改，那么你会发现页面顶部的那根横线依然是原来的黑色。定位其代码，找到 `base.styl` 中 `$headband-bg = $black-deep;`，同样进行修改为 `$theme-color` 即可。

### 修改背景颜色

Pieces 默认背景颜色与纯白非常接近，是 `#f5f7f9`，我想修改为另外的颜色。利用搜索，很快定位到代码：
`themes/next/source/css/_variables/Pisces.styl` 中，`$body-bg-color = #f5f7f9;`。
修改其值即可。
