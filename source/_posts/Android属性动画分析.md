---
title: Android属性动画分析
date: 2017-11-25 10:54:26
tags:
	- Android
	- 动画
category: Android
comments: true
---

### 属性动画分析概略
Android 动画分三类:  
`Drawable Animation`:几帧图循环播放；  
`View Animation` : 只有基本的 `缩放`，`平移`，`旋转`，`渐变`，通过对 `Canvas` 操作来实现。只在绘制层做了动画假象，实际属性并未改变  
`ValueAnimator`:属性动画，可以实现更平滑的动画和对属性的变更。

我将从以下几个方面分析 属性动画的实现原理  
1. 通过常用 `ObjectAnimator.of..` 方法切入,深入分析 `PropertyValueHolder`,`Property`,`Keyframe`的实现原理和作用。    
2. 通过对属性动画的启动方式的解析 代入 Android 新的渲染方式(黄油计划)  
3. 属性动画其他常用API的使用  
4. `ValueAnimator`和`ObjectAnimator`的关系以及如何选择  
5. 高级的属性动画自定义方式。例如自定义`PropertyValueHolder`和自定义`keygrame`实现属性动画效果。  

针对源码分析,前三篇文章会涉及的多一些，也更能体会到Android属性动画设计的巧妙和灵活之处。后几篇文章着重应用。

**作为一个有责任心的程序员**，我希望通过这几篇文章，能帮助小伙伴们更深入理解Android属性动画，而不仅仅局限在使用上。
