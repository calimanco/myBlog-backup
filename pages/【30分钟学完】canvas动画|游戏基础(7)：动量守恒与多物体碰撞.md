## 前言

一路沿着本系列教程学习的朋友可能会发现，前面教程中都尽量避免提及质量的概念，很多运动概念也时刻提醒大家这不是真实的物体运动。因为真实的物体运动其实跟质量都是密不可分的，而且质量的引入自然必须提及力学概念，所以为了不内容冗余才忽略了质量。  
从本篇开始，将会正式引入物理力学概念，给每个物体赋予质量概念，为了更加真实的模拟现实环境的物体运动。  
阅读本篇前请先打好前面的基础。  
本人能力有限，欢迎牛人共同讨论，批评指正。

## 动量与动量守恒

> 【科普】一般而言，一个物体的动量指的是这个物体在它运动方向上保持运动的趋势。动量实际上是牛顿第一定律的一个推论。

动量即是“物体运动的量”，是物体的质量和速度的乘积，是矢量，能够反应出运动的效果，一般用p表示。举个例子，低速运动的重物，跟高速运动的子弹，拥有相同的威力。  

    p = m * v

> 【科普】动量是守恒量。动量守恒定律表示为：一个系统不受外力或者所受外力之和为零，这个系统中所有物体的总动量保持不变。它的一个推论为：在没有外力干预的情况下，任何系统的质心都将保持匀速直线运动或静止状态不变。动量守恒定律可由机械能对空间平移对称性推出。

动量守恒即系统在碰撞前的总动量等于系统在碰撞后的总动量。其中的系统简单理解就是物体的集合。在可以忽略碰撞以外的因素时，动量是守恒的。  

    (m0 * v0) + (m1 * v1) = (m0 * v0Final) + (m1 * v1Final)

这条公式是我们计算碰撞后速度的基础，现在只要推导出末速度v0Final和v1Final的公式，就可以应用到我们的模拟碰撞的编程动画中。推导过程如下：

![推导末速度][1]

其实推导过程不重要，只要记得结论：  

```javascript
v1Final = (2 * m0 * v0) + v1 * (m1 - m0) / (m0 + m1)
v0Final = (2 * m1 * v1) - v0 * (m0 - m1) / (m0 + m1)
// 二者可直接转换
v1Final = (v0 - v1) + v0Final
```

## 单轴碰撞

我们开始使用前面推导出的公式，先来个最简单的单轴碰撞例子，这里演示了两个球相撞的效果，mass定义了他们的质量，由于他们初始速度相同，所以依据动量守恒碰撞后ball0的速度变为-1/3，而ball1的速度变为5/3。  
这里有个细节，碰撞时可能出现球已经重叠的情况，这个例子只是简单将末速度加给碰撞后的球，用以弹开他们，但仔细想想就知道这是很不严谨的做法。  

![单轴碰撞][2]

```javascript
/**
 * 单轴碰撞
 * */
window.onload = function () {
  const canvas = document.getElementById('canvas');
  const context = canvas.getContext('2d');
  const ball0 = new Ball();
  const ball1 = new Ball();
  // 定义ball0的属性
  ball0.mass = 2;
  ball0.x = 50;
  ball0.y = canvas.height / 2;
  ball0.vx = 1;
  // 定义ball1的属性
  ball1.mass = 1;
  ball1.x = 300;
  ball1.y = canvas.height / 2;
  ball1.vx = -1;

  (function drawFrame() {
    window.requestAnimationFrame(drawFrame, canvas);
    context.clearRect(0, 0, canvas.width, canvas.height);
    // 移动两个物体的位置
    ball0.x += ball0.vx;
    ball1.x += ball1.vx;
    const dist = ball1.x - ball0.x;
    // 碰撞检测
    if (Math.abs(dist) < ball0.radius + ball1.radius) {
      // 运用动量守恒计算碰撞后速度
      const vxTotal = ball0.vx - ball1.vx;
      ball0.vx = ((ball0.mass - ball1.mass) * ball0.vx + 2 * ball1.mass * ball1.vx) / (ball0.mass + ball1.mass);
      ball1.vx = vxTotal + ball0.vx;
      // 将速度加到两物体的位置上实现弹开
      ball0.x += ball0.vx;
      ball1.y += ball1.vx;
    }
    // 绘制两球
    ball0.draw(context);
    ball1.draw(context);
  }());
};
```

## 双轴碰撞

现实情况很少会出现单轴碰撞，如果两个轴上都有速度，处理起来会比较麻烦，把速度分解出来再代入动量守恒公式，这里运用到上一篇中关于坐标旋转的知识。  

![双轴碰撞][3]  

基本思路：  

1.  使用旋转公式，旋转整个系统，将两物体的中心连线置为水平场景；
2.  求出物体x轴上的速度；
3.  使用动量守恒计算x轴上的碰撞后速度；
4.  再旋转回来。


[1]: https://nimokuri.github.io/myBlog-backup/assets/【30分钟学完】canvas动画|游戏基础(7)：动量守恒与多物体碰撞/1.png

[2]: https://nimokuri.github.io/myBlog-backup/assets/【30分钟学完】canvas动画|游戏基础(7)：动量守恒与多物体碰撞/2.png

[3]: https://nimokuri.github.io/myBlog-backup/assets/【30分钟学完】canvas动画|游戏基础(7)：动量守恒与多物体碰撞/3.png
