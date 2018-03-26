## 前言

本系列前几篇中常出现物体跑到画布外的情况，本篇就是为了解决这个问题。  
阅读本篇前请先打好前面的基础。  
本人能力有限，欢迎牛人共同讨论，批评指正。  

## 越界检测

假定物体是个圆形，如图其圆心坐标即是物体的x轴和y轴坐标。  
越界是常见的场景，一般会有两种场景的越界：一是整个物体移出区域，二是物体接触到区域边界。我们以画布边界为例进行讨论，示例中矩形边界即是：  

```javascript
let top = 0;
let bottom = canvas.height;
let left = 0;
let right = canvas.width;
```

![边界][1]

### 整个物体移出区域

要整个物体离开范围才算越界，则可得越界条件如下，以下任何一项为true即可判定越界。  

```javascript
// 右侧越界
object.x - object.width/2 > right
// 左侧越界
object.x + object.width/2 < left
// 上部越界
object.y + object.height/2 < top
// 下部越界
object.y - object.height/2 > bottom
```

### 物体接触到区域边界

物体接触到区域边界就算越界，则可得越界条件如下，以下任何一项为true即可判定越界。  

```javascript
// 右侧越界
object.x + object.width/2 > right
// 左侧越界
object.x - object.width/2 < left
// 上部越界
object.y - object.height/2 < top
// 下部越界
object.y + object.height/2 > bottom
```

## 越界了该怎么办

搞明白越界条件后，接下来讨论越界之后的处理办法，一般是一下四种。  

### 将物体移除

这是最简单的处理办法，属于整个物体移出区域才算越界的情况。  
下面的例子会先批量创建ball，保存在balls数组里，每次动画循环都会遍历这个数组，依次输入draw()函数，改变ball的位置并检测是否越界。下面只列出draw()函数的代码。  
完整示例：[清除越界圆][2]

```javascript
function draw(ball, pos) {
  // 依据球的速度改变球的位置
  ball.x += ball.vx;
  ball.y += ball.vy;
  // 检查是否越界
  if (ball.x - ball.radius > canvas.width || ball.x + ball.radius < 0 || ball.y - ball.radius > canvas.height || ball.y + ball.radius < 0) {
    // 在数组中清除越界的球
    balls.splice(pos, 1);
    // 打印提示
    if (balls.length > 0) {
      log.value += `Removed ${ball.id}\n`;
      log.scrollTop = log.scrollHeight;
    } else {
      log.value += 'All gone!\n';
    }
  }
  // 画球
  ball.draw(context);
}
```

### 将其物体置回边界内

属于整个物体移出区域才算越界的情况。  
下面的例子也是把创建的ball保存在balls数组里，但ball的初始位置都是画布中间的下部，如果检测到有ball越界，则会重置ball的位置。下面只列出draw()函数的代码。  
完整示例：[彩色喷泉][3]

```javascript
function draw(ball) {
  // 依据球的速度改变球的位置，这里包含了伪重力
  ball.vy += gravity;
  ball.x += ball.vx;
  ball.y += ball.vy;
  // 检测是否越界
  if (ball.x - ball.radius > canvas.width || ball.x + ball.radius < 0 || ball.y - ball.radius > canvas.height || ball.y + ball.radius < 0) {
    // 重置ball的位置
    ball.x = canvas.width / 2;
    ball.y = canvas.height;
    // 重置ball的速度
    ball.vx = Math.random() * 6 - 3;
    ball.vy = Math.random() * -10 - 10;
    // 打印提示
    log.value = `Reset ${ball.id}\n`;
  }
  // 画球
  ball.draw(context);
}
```

### 屏幕环绕

属于整个物体移出区域才算越界的情况。  
屏幕环绕就是让同一个物体出现在边界内的另一个位置，如果一个物体从屏幕左侧移出，它就会在屏幕右侧再次出现，反之亦然，上下也是同理。  
这里比前面的要稍微复杂的判断物体跃的是那边的界，伪代码如下：  

```javascript
if(object.x - object.width/2 > right){
	object.x = left - object.widht/2;
}else if(object.x + object.width/2 < left){
	object.x = right + object.width/2;
}
if(object.y - object.height/2 > bottom){
	object.y = top - object.height/2;
}else if(object.y + object.height/2 < top){
	obejct.y = bottom + object.height/2;
}
```

### 反弹（粗略版）

这是较复杂的一种情况，属于物体接触到区域边界就算越界的情况。基本思路：  

1.  检查物体是否越过任意边界；
2.  如果发生越界， 立即将物体置回边界；
3.  反转物体的速度向量的方向。

下面的示例是一个ball在画布内移动，撞到边界就反弹，反弹核心代码如下。  
完整示例：[反弹球（粗略版）][4]

```javascript
if (ball.x + ball.radius > right) {
  ball.x = right - ball.radius;
  vx *= -1;
} else if (ball.x - ball.radius < left) {
  ball.x = left + ball.radius;
  vx *= -1;
}
if (ball.y + ball.radius > bottom) {
  ball.y = bottom - ball.radius;
  vy *= -1;
} else if (ball.y - ball.radius < top) {
  ball.y = top + ball.radius;
  vy *= -1;
}
```

### 反弹（完美版）

咋看似乎效果不错，但仔细想想，我们这样将物体置回边界的做法是准确的吗？  
答案是否定的，理想反弹与实际反弹是不同的，如下图：  

![理想反弹与实际反弹][5]  

从图中我们可以清除的知道，ball实际上是不太可能会在理想反弹点反弹的，因为如果速度过大，计算位置时ball已经越过“理想反弹点”到达“实际反弹点”，而我们如果只是将ball的x轴坐标简单粗暴移到边界上，那还是不可能是“理想反弹点”，也就是说这种反弹方法不准确。  
那么，完美反弹的思路就明确了，我们需要找到“理想反弹点”，并将ball置到该点。  
[1]: https://nimokuri.github.io/myBlog-backup/assets/【30分钟学完】canvas动画|游戏基础(4)：边界与碰撞/1.png

[2]: https://nimokuri.github.io/H5Learning-animationDemo/part5/01-removal.html

[3]: https://nimokuri.github.io/H5Learning-animationDemo/part5/02-fountain.html

[4]: https://nimokuri.github.io/H5Learning-animationDemo/part5/04-bouncing-1.html

[5]:https://nimokuri.github.io/myBlog-backup/assets/【30分钟学完】canvas动画|游戏基础(4)：边界与碰撞/2.png
