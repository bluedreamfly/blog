---
title: 谈谈CSS Grid Layout
date: 2017-04-16 20:55:55
categories: ['技术', '前端', 'CSS']
---

&emsp;&emsp;一谈到CSS布局，我想大家都有各种各样的方式，比如table、float、position、inline-block，还有flexbox。我想也在布局上遇到了很多问题，而这些布局方式除了flexbox我想都是一些hack，它们本质上并不是用来布局的。但是flexbox是属于一维布局，对于二维布局可能相对还是要麻烦一点。今天我们将要介绍一种新的特别强大的布局方式————CSS Grid(网格布局)。

&emsp;&emsp;我们以前可能或多或少听过这种布局方式，因为CSS Grid早在2011就已经发布了草案。我们都知道W3C发布的一些规范，到浏览器实现其实是需要很长的一段时间，如果是一家浏览器厂商还好，但是前端领域最头疼的无非是兼容性问题，所以我们只能等。而现在比较新版本的浏览器已经开始支持这种布局方式了，是时候可以开始去探索它强大的特性了。下面我们看下它的浏览器兼容情况。

![](http://oofjmuxr0.bkt.clouddn.com/css_grid_compatible.jpeg)

&emsp;&emsp;貌似情况不容乐观，但至少我们有可以测试的地方了。我的想法是新技术有时间先倒腾一下，先了解它的优势，后面真正用到的时候也能更好的选择。所以不要说浏览器支持这么差，学了也没用，我觉得很多新技术的思想还是很好的，可能在实际业务中我们用不到，但是我们却可以借鉴它的想法，我觉得这才是最重要的。废话不多说了，开始我们的CSS Grid之旅吧。

&emsp;&emsp;首先看个[demo](http://demo.51hzh.me/css-grid)，感受下它的强大。

### 术语
&emsp;&emsp; 那我们先来介绍一些网格中的术语：grid lines（网格线）、grid track（网格轨迹）、grid cell（网格）及grid area（网格区域）

#### grid lines
网格线是网格的水平和垂直分界线。就是下图的line1, line2这些线组成的。

![](http://oofjmuxr0.bkt.clouddn.com/grid-lines.png)

#### grid track
网格轨迹其实是网格行（row）和列（column）的别称。

#### grid cell 
这个其实就没什么好说的，这其实就是相邻的网格线围绕成的一个单元格。其实跟table中的单元格意思是一样的。

#### grid area 
网格区域由一个或多个邻接网格单元组成，也就是说网格单元是特殊的网格区域。

那么接下来我们就一步步来实现一个网格布局系统

```css
.container {
	display: grid;
		//grid 块级网格布局
		//inline-grid: 行级网格布局
		//subgrid
	grid-template-columns: 200px auto 200px;
}
```










