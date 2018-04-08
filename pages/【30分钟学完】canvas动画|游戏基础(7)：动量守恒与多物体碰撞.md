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

> 【PS】这里有个细节，碰撞时可能出现球已经重叠的情况，这个例子只是简单将末速度加给碰撞后的球，用以弹开他们，这是不严谨但有效的做法。  

完整示例：[单轴碰撞][4]

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

1.  使用旋转公式，以其中一个物体为原电，旋转整个系统，将两物体的中心连线置为水平场景；
2.  求出物体x轴上的速度；
3.  使用动量守恒计算x轴上的碰撞后速度；
4.  再旋转回来。

示例是两个随机初始速度的球在空间内碰撞，碰到边界也会反弹，由于代码量较大，这里只截取部分核心代码：  
注意：旋转是以ball0为原点进行的，也就是说旋转中的所有位置和速度都是相对于ball0的，所有回旋后的位置和速度需要转换成相对于相对区域位置。  
完整示例：[双轴碰撞][5]

```javascript
// 坐标旋转函数
function rotate(x, y, sin, cos, reverse) {
  return {
    x: (reverse) ? (x * cos + y * sin) : (x * cos - y * sin),
    y: (reverse) ? (y * cos - x * sin) : (y * cos + x * sin),
  };
}
// 检查碰撞
function checkCollision() {
  const dx = ball1.x - ball0.x;
  const dy = ball1.y - ball0.y;
  const dist = Math.sqrt(dx ** 2 + dy ** 2);

  // 基于距离的碰撞检测
  if (dist < ball0.radius + ball1.radius) {
    // 以ball0为中心点旋转
    const angle = Math.atan2(dy, dx);
    const sin = Math.sin(angle);
    const cos = Math.cos(angle);
    // ball0在中心点
    const pos0 = {
      x: 0,
      y: 0,
    };
    // 依据ball1与ball0的相对距离计算旋转后的坐标（反向）
    const pos1 = rotate(dx, dy, sin, cos, true);
    // 旋转ball0的速度（反向）
    const vel0 = rotate(ball0.vx, ball0.vy, sin, cos, true);
    // 旋转ball1的速度（反向）
    const vel1 = rotate(ball1.vx, ball1.vy, sin, cos, true);
    // 计算相对速度
    const vxTotal = vel0.x - vel1.x;
    // 计算相撞后速度
    vel0.x = ((ball0.mass - ball1.mass) * vel0.x + 2 * ball1.mass * vel1.x) / (ball0.mass + ball1.mass);
    vel1.x = vxTotal + vel0.x;
    // 计算相撞后位置
    pos0.x += vel0.x;
    pos1.x += vel1.x;

    // 回旋位置
    const pos0F = rotate(pos0.x, pos0.y, sin, cos, false);
    const pos1F = rotate(pos1.x, pos1.y, sin, cos, false);

    // 将相对ball0位置转换为相对区域位置
    ball1.x = ball0.x + pos1F.x;
    ball1.y = ball0.y + pos1F.y;
    ball0.x += pos0F.x;
    ball0.y += pos0F.y;
    // 回旋速度
    const vel0F = rotate(vel0.x, vel0.y, sin, cos, false);
    const vel1F = rotate(vel1.x, vel1.y, sin, cos, false);
    ball0.vx = vel0F.x;
    ball0.vy = vel0F.y;
    ball1.vx = vel1F.x;
    ball1.vy = vel1F.y;
  }
}
```

## 多物体碰撞

加入多个物体，只是把两个物体的碰撞检测，改变成所有物体两两间做碰撞检测。

基本思路：  

1.  先遍历一次物体集，让物体移动并处理边界碰撞；
2.  再遍历一次物体集，两两物体做碰撞检测并求出碰撞后的速度和位置；
3.  最后一次遍历物体集，绘制他们。

依据这个思路我们得到了这样一个示例，球的质量、大小和初始速度都是随机的，碰撞代码基本和前面是一样的。  
完整示例：[多物体碰撞（无处理重叠）][6]  

仔细观察示例，会发现这里会出现一个问题：小球会重叠到一起并且无法分离。这是由如下原因造成的：  

-   程序依照三个小球的速度移动他们；
-   程序检测ball0和ball1，ball0和ball2，发现他们并没有碰撞；
-   程序检测ball1和ball2。因为他们发生了碰撞，所以他们的速度和位置都要重新计算，然后弹开。但这不巧让ball1和ball0 接触上了。然而，由于这一组合已经过检测，所以忽略了这一事实；
-   在下一轮循环中，程序依然按照他们的速度移动小球。这样就使得ball0和ball1更加靠近了；
-   现在程序检测到ball0和ball1碰撞了，重新计算速度和位置后，想要将他们分开，却会出现无法完全分开的情况，就卡到了一起。

> 【PS】为什么无法完全分开？因为我们分开两物体的做法是将新速度加到新位置上，如果旧位置已经重叠，那就永远无法分离了。  

改变分开两物体的处理办法就能解决这个问题，这里有个较为简单但不是很精确的办法：  

1.  先求给出总速度绝对值；
2.  再求出重叠部分的长度；
3.  以相撞后速度在总速度的比例移开两个物体。

完整示例：[多物体碰撞][7]  
改造后核心代码如下：  

```javascript
function checkCollision(ball0, ball1) {
  const dx = ball1.x - ball0.x;
  const dy = ball1.y - ball0.y;
  const dist = Math.sqrt(dx ** 2 + dy ** 2);

  // 基于距离的碰撞检测
  if (dist < ball0.radius + ball1.radius) {
    // 以ball0为中心点旋转
    const angle = Math.atan2(dy, dx);
    const sin = Math.sin(angle);
    const cos = Math.cos(angle);
    // ball0在中心点
    const pos0 = {
      x: 0,
      y: 0,
    };
    // 依据ball1与ball0的相对距离计算旋转后的坐标（反向）
    const pos1 = rotate(dx, dy, sin, cos, true);
    // 旋转ball0的速度（反向）
    const vel0 = rotate(ball0.vx, ball0.vy, sin, cos, true);
    // 旋转ball1的速度（反向）
    const vel1 = rotate(ball1.vx, ball1.vy, sin, cos, true);
    // 计算相对速度
    const vxTotal = vel0.x - vel1.x;
    // 计算相撞后速度
    vel0.x = ((ball0.mass - ball1.mass) * vel0.x + 2 * ball1.mass * vel1.x) / (ball0.mass + ball1.mass);
    vel1.x = vxTotal + vel0.x;
    // 计算出绝对速度和重叠量，分离避免物体重叠
    const absV = Math.abs(vel0.x) + Math.abs(vel1.x);
    const overlap = (ball0.radius + ball1.radius) - Math.abs(pos0.x - pos1.x);
    pos0.x += vel0.x / absV * overlap;
    pos1.x += vel1.x / absV * overlap;

    // 回旋位置
    const pos0F = rotate(pos0.x, pos0.y, sin, cos, false);
    const pos1F = rotate(pos1.x, pos1.y, sin, cos, false);

    // 将相对ball0位置转换为相对区域位置
    ball1.x = ball0.x + pos1F.x;
    ball1.y = ball0.y + pos1F.y;
    ball0.x += pos0F.x;
    ball0.y += pos0F.y;
    // 回旋速度
    const vel0F = rotate(vel0.x, vel0.y, sin, cos, false);
    const vel1F = rotate(vel1.x, vel1.y, sin, cos, false);
    ball0.vx = vel0F.x;
    ball0.vy = vel0F.y;
    ball1.vx = vel1F.x;
    ball1.vy = vel1F.y;
  }
}
```

[1]: https://nimokuri.github.io/myBlog-backup/assets/【30分钟学完】canvas动画|游戏基础(7)：动量守恒与多物体碰撞/1.png

[2]: https://nimokuri.github.io/myBlog-backup/assets/【30分钟学完】canvas动画|游戏基础(7)：动量守恒与多物体碰撞/2.png

[3]: https://nimokuri.github.io/myBlog-backup/assets/【30分钟学完】canvas动画|游戏基础(7)：动量守恒与多物体碰撞/3.png

[4]: https://nimokuri.github.io/H5Learning-animationDemo/part10/02-billiard-2.html

[5]: https://nimokuri.github.io/H5Learning-animationDemo/part10/04-billiard-4.html

[6]: https://nimokuri.github.io/H5Learning-animationDemo/part10/05-multi-billiard-1

[7]: https://nimokuri.github.io/H5Learning-animationDemo/part10/06-multi-billiard-2
