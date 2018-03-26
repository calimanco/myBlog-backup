## 前言

上一篇基本已经对canvas打好了基础，本篇主要将引入经典物理概念速度和加速度，探讨他们在编程动画中的应用。
在阅读之前请先自行了解速度和加速度的基础知识，以及向量与向量加法。
本人能力有限，欢迎牛人共同讨论，批评指正。

## 速度

> 【科普】速度是描述物体运动快慢和方向的物理量。物理学中提到物体的速度通常是指其瞬时速度。速度在国际单位制中的单位是米每秒，国际符号是m/s，中文符号是米/秒。相对论框架中，物体的速度上限是光速。

在动画编程中，速度是最基础的要素，在本系列教程的第2篇讲到三角函数时就有所体现。  

### 动画编程中的速度

见下图，我们这里要讨论的计算机动画中的速度跟物理学上概念相似，都是矢量，也就是**既有大小又有方向**，而方向的体现就是其值的正负，回顾第1篇中讲到的坐标系，沿着正半轴运动速度就是正，沿着负半轴运动速度就是负。  
另外一点不同就是单位，不一定会以时间为单位，可能是**以帧为单位**，比如“像素/帧”。  
也正因为速度是矢量，那任何一个速度都可以分解为x轴和y轴上的速度，这就是编程动画基本思想。  

![速度分解图][1]  

### 实例应用

将系列第2篇的“一个会跟踪鼠标位置的箭头”改造成“跟随鼠标的箭头”。代码很基础，看注释就行，基本思路：

1.  计算目标点与物体的夹角；
2.  依据夹角分解速度到x轴和y轴；
3.  将分解后的速度加到物体的两轴位置上；
4.  重新绘制物体。

特别说明，因为这个例子中的动画循环是基于帧，所以速度单位是像素每帧。  
完整例子：[跟随鼠标的箭头][2]

```javascript
/**
 * 跟随鼠标的箭头
 * */
window.onload = function () {
  const canvas = document.getElementById('canvas');
  const context = canvas.getContext('2d');
  const mouse = utils.captureMouse(canvas);
  const arrow = new Arrow();
  // 设定速度
  const speed = 3;

  (function drawFrame() {
    window.requestAnimationFrame(drawFrame, canvas);
    context.clearRect(0, 0, canvas.width, canvas.height);

    // 计算鼠标与箭头的相对距离
    const dx = mouse.x - arrow.x;
    const dy = mouse.y - arrow.y;
    // 求箭头指向鼠标的夹角
    const angle = Math.atan2(dy, dx);
    // 将速度分解到x轴和y轴
    const vx = Math.cos(angle) * speed;
    const vy = Math.sin(angle) * speed;
    // 设置箭头的角度
    arrow.rotation = angle;
    // 将分解后的速度加到箭头的两轴位置上
    arrow.x += vx;
    arrow.y += vy;
    // 重新绘制箭头
    arrow.draw(context);
  }());
};
```

[1]: https://nimokuri.github.io/myBlog-backup/assets/【30分钟学完】canvas动画|游戏基础(3)：速度与加速度/1.png

[2]: https://nimokuri.github.io/H5Learning-animationDemo/part4/04-follow-mouse.html
