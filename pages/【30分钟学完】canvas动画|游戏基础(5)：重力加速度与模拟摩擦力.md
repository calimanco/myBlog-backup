## 前言

解决运动和碰撞问题后，我们为了让运动环境更加自然，需要加入一些环境因子，比如常见的重力加速度和模拟摩擦力。  
阅读本篇前请先打好前面的基础。  
本人能力有限，欢迎牛人共同讨论，批评指正。  

## 重力加速度

> 【科普】重力加速度是一个物体受重力作用的情况下所具有的加速度。也叫自由落体加速度，用g表示。方向竖直向下。通常指地面附近物体受地球引力作用在真空中下落的加速度，记为g。为了便于计算，其近似标准值通常取为980厘米/秒的二次方或9.8米/秒的二次方。

真实的物体是有质量的，所以其重力加速度是由于重力产生，而我们计算机中的抽象物体并没有质量，所有也不存在重力一说，我们这里说的重力加速度只是借用了物理上的概念，实际上是人为定义的一个**方向指向y轴正半轴的加速度**。  
其实实现起来很简单，就是设定一个为正的加速度，每次绘制都加到物体的y轴速度上。  
下面的示例是一个ball，它会受重力加速度gravity而自动下落，你可以使用键盘的上、下、左、右改变其四个方向上的加速度。核心代码如下：  
完整示例：[重力加速度演示][1]

```javascipt
(function drawFrame() {
  window.requestAnimationFrame(drawFrame, canvas);
  context.clearRect(0, 0, canvas.width, canvas.height);

  vx += ax;
  vy += ay;
  vy += gravity;
  ball.x += vx;
  ball.y += vy;
  ball.draw(context);
}());
```

## 模拟摩擦力

> 【科普】阻碍物体相对运动（或相对运动趋势）的力叫做摩擦力。摩擦力的方向与物体相对运动（或相对运动趋势）的方向相反。一个物体在另一个物体表面发生滑动时，接触面间产生阻碍它们相对运动的摩擦，称为滑动摩擦。滑动摩擦力的大小与接触面的粗糙程度的大小和压力大小有关。压力越大，物体接触面越粗糙，产生的滑动摩擦力就越大。

之前的例子中有一些非常不自然的场景，比如[跟随鼠标的箭头][2]，由于加速度始终存在，导致运动永远不可能停止，而在现实中（太空例外），由于存在各种摩擦力的关系，这是不可能发生的情况。  
计算机中没有摩擦力，我们只是借鉴物理中的概念模拟一个**模拟摩擦力**，请记住这个并不是物理意义的力。  

> 【定义】模拟摩擦力是人为规定的值，定义和滑动摩擦力相似都与运动方向相反的量，将物体速度削减到0为止，不会改变运动方向。

注意：根据定义只能将物体的速率减去与一定大小的值，而不能分别在x, y轴上减小速度向量。如果物体正沿着某个角度运动，就会出现物体在某条轴的速度降为零，而继续在另一条轴上运动的奇怪现象。    

### 正确做法

我们将模拟摩擦力用变量friction表示，示例会演示随机速度的ball从运动到停止的过程，核心代码如下，基本思路：  

>【科普】速度和速率是两个不同的概念。速度是矢量，具有大小和方向；速率则纯粹指物体运动的快慢，是标量，没有方向。

1. 将vx与vy平方后求和，再开方求出速率；通过计算Math.atan2(vy, vx)获得角度；
2. 从速率减去模拟摩擦力，但不要让速率变为负数；
3. 通过正余弦函数将和速率分解为x轴和y轴上的速度。

完整示例：[模拟摩擦力正确计算][3]

```javascript
(function drawFrame() {
  window.requestAnimationFrame(drawFrame, canvas);
  context.clearRect(0, 0, canvas.width, canvas.height);
  // 先求速率
  let speed = Math.sqrt(vx ** 2 + vy ** 2);
  // 算出角度
  const angle = Math.atan2(vy, vx);
  // 判断运动是否停止
  if (speed > friction) {
    // 没有停止则减去模拟摩擦力
    speed -= friction;
  } else {
    speed = 0;
  }
  // 重新分解为x轴和y轴上的速度
  vx = Math.cos(angle) * speed;
  vy = Math.sin(angle) * speed;
  ball.x += vx;
  ball.y += vy;
  ball.draw(context);
}());
```

### 简便做法

正确的做法十分繁琐，是个合成分解再合成的过程，这样对计算资源的消耗是比较大的，但我们也许并不需要这么精确的做法，只要每次将各个方向的速度乘以一个0~1之间的数就能简单模拟出摩擦力的效果。因此我们定义了**模拟摩擦力系数**。  

> 【定义】模拟摩擦力系数是人为规定的值，会在物体运动时不断比例减少各个方向上的速度，使各个方向的速度无限接近于0。

示例由上面的正确做法改造而来，friction被定义为模拟摩擦力系数，指为0.9，只要运动都将x轴和y轴方向的速度乘以这个值即可，减少了大量操作。核心代码如下：  
完整示例：[模拟摩擦力正确计算][3]  
注意：这里有一个细节，速度不断乘以系数会导致速度**无限接近但不等于0**，为了避免做无意义的计算，可以先判断速度是否已经小到肉眼不可见的值，以提高性能。  

```javascript
(function drawFrame() {
  window.requestAnimationFrame(drawFrame, canvas);
  context.clearRect(0, 0, canvas.width, canvas.height);
  // 判断速度大小以减少不必要的计算
  if (Math.abs(vx) > 0.001) {
    // 减少速度
    vx *= friction;
    ball.x += vx;
  }
  if (Math.abs(vy) > 0.001) {
    vy *= friction;
    ball.y += vy;
  }
  ball.draw(context);
}());
```

[1]: https://nimokuri.github.io/H5Learning-animationDemo/part4/09-gravity.html

[2]: https://nimokuri.github.io/H5Learning-animationDemo/part4/04-follow-mouse.html

[3]: https://nimokuri.github.io/H5Learning-animationDemo/part5/06-friction-1.html
