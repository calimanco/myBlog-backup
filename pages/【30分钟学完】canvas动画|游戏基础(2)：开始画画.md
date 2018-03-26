## 前言 ##
上篇主要是理论的概述，本篇会多些实践，来讲讲canvas的基础用法，并包含一些基础三角函数的应用，推荐没有canvas基础的朋友阅读，熟悉的朋友可以跳过。  
本人能力有限，欢迎牛人共同讨论，批评指正。  

## 一起来画画吧 ##
canvas的API有很多，如果一一列举30分钟你是绝对看不完的，而且怎么流水账还不如自己去看文档呢（笑），本教程的思路是用实例一步一步从无到有讲解基础用法。[canvas相关文档][1]  

### 准备工作 ###
1. 布置画布：通过添加<canvas>标签，添加canvas元素
2. 获取画布：通过<canvas>标签的id，获得canvas对象
3. 获得画笔：通过canvas对象的getContext("2d")方法，获得2D环境

``` html
<canvas id="canvas" width="400" height="400"></canvas>
```

``` javascript
const canvas = document.getElementById('canvas');
const context = canvas.getContext('2d');
```

### 画个箭头 ###
首先我们来画个**红边黄底的箭头**，使用面向对象的代码组织方式，全部代码如下。  
类名为Arrow。它拥有x轴坐标、y轴坐标、底的颜色color、旋转弧度rotation四个属性。  
实例方法是draw()，它需要一个context对象作为参数，就是准备工作里的context，它就相当于是画笔，这里其实是类似依赖注入的过程，将canvas的画笔交给实例的draw()方法，实例用这个画笔去画出箭头，绘画过程见代码注释。特别注意以下几点：
- beginPath()方法调用后moveTo()和lineTo移动坐标是**相对与beginPath()时画笔的坐标的**，可以理解成画笔自带一个坐标系，它可以旋转和在画布上移动，绘制工作的坐标都是属于这个坐标系的；
- beginPath()是绘制设置状态的起始点，它之后代码设置的绘制状态的作用域结束于绘制方法stroke()、fill()或者closePath()；
- save()的作用是保存笔的状态，因为一个画布的笔只有一支，会在不同对象中传递，为了不污染后续的画就应该先保存，画完再restore()还原；
- **<canvas>本身是透明的**，可以使用CSS给它个背景，例子中普遍使用白色背景。

``` javascript
/**
 * 箭头类
 * @class Representing a arrow.
 */
/* eslint no-unused-vars: ["error", { "varsIgnorePattern": "Arrow" }] */
class Arrow {
  /**
    * Create a arrow.
    */
  constructor() {
    this.x = 0;
    this.y = 0;
    this.color = '#ffff00';
    this.rotation = 0;
  }
  /**
   * Draw the arrow.
   * @param {Object} _context - The canvas context.
   */
  draw(_context) {
    const context = _context;
    // 会先保存画笔状态
    context.save();
    // 移动画笔
    context.translate(this.x, this.y);
    // 旋转画笔
    context.rotate(this.rotation);
    // 设置线条宽度
    context.lineWidth = 2;
    // 设置线条颜色
    context.strokeStyle = '#ff0000';
    // 设置填充颜色
    context.fillStyle = this.color;
    // 开始路径
    context.beginPath();
    // 将笔移动到相对位置
    context.moveTo(-50, -25);
    // 画线到相对位置
    context.lineTo(0, -25);
    context.lineTo(0, -50);
    context.lineTo(50, 0);
    context.lineTo(0, 50);
    context.lineTo(0, 25);
    context.lineTo(-50, 25);
    context.lineTo(-50, -25);
    // 闭合路径
    context.closePath();
    // 填充路径包围区
    context.fill();
    // 绘制路径
    context.stroke();
    // 载入保存的笔信息
    context.restore();
  }
}
```
同理我们还可以画点其他的，比如一个圆[ball.js][5]，稍微多些参数，慢慢理解。  
成品的效果可以先看这个（稍微剧透）：[一个会跟踪鼠标位置的箭头][3]  

### 加入循环动起来 ###
现在我们已经掌握了画画的基本功，并且可以画箭头[arrow.js][4]和圆[ball.js][5]，然而这样只是静止画，接下来我们需要一个循环，不断的执行擦除和重画的工作才能实现帧动画。
下面这段代码的中绘图函数drawFrame被立即执行，并递归调用自身，你将会在大部分例子中看到。  
循环原理上一篇已经说明，不再重复。这里要说明的是clearRect()，这个函数接受一个矩形坐标，也就是（x轴坐标，y轴坐标，矩形宽度，矩形高度），用于清除矩形区域内的画。  
例子里直接是清除了整个画布，但这不是绝对的，刷不刷新，是局部刷新还是全部刷新，都需要灵活处理。
这里有个不刷新的例子  

``` javascript
(function drawFrame() {
  // 类似setTimeout的操作
  window.requestAnimationFrame(drawFrame, canvas);
  // 将画布擦干净
  context.clearRect(0, 0, canvas.width, canvas.height);
  // ...继续你的作画
}());
```

### 给它点动力 ###
现在画面已经是在不断的重绘，但为什么还是静止的呢？因为每一次刷新都没有改变要画的内容。  
那我们就给它一个目标吧，这样它才能动起来，比如就让箭头始终指向鼠标。  
下面是核心代码，主要目的就是求出每帧arrow的旋转角度，这里使用的工具类mouse会实时返回鼠标的x，y轴坐标，封装原理上一篇已经讲过，根据这鼠标的坐标和arrow的坐标，即可得到鼠标的相对于arrow的距离dx和dy，如下图：
![图片描述][6]

而arrow的旋转角度即可以通过dx和dy使用反正切函数得到，这里需要注意几点：
- 仔细看上面代码中arrow的绘制过程，可知其原点是在中心位置的，所以刚好旋转角度就是画笔的旋转角度；
- dx和dy是鼠标相对与arrow的坐标，所以图中把坐标系挪动箭头中心是没毛病的；
- 用atan2，而不是atan，是因为tan值本来就可能是重复的，比如-1/2和1/(-2)两个都是-0.5，无法区分象限，而atan2就可以区分开。

完整实例：[一个会跟踪鼠标位置的箭头][3]
``` javascript
window.onload = function () {
  const canvas = document.getElementById('canvas');
  const context = canvas.getContext('2d');
  const mouse = utils.captureMouse(canvas);
  const arrow = new Arrow();

  arrow.x = canvas.width / 2;
  arrow.y = canvas.height / 2;

  (function drawFrame() {
    window.requestAnimationFrame(drawFrame, canvas);
    context.clearRect(0, 0, canvas.width, canvas.height);
    const dx = mouse.x - arrow.x;
    const dy = mouse.y - arrow.y;

    arrow.rotation = Math.atan2(dy, dx);
    arrow.draw(context);
  }());
};
```

## 三角函数 ##
### 上下运动 ###
终于顺利过渡到三角函数的话题（笑）。三角函数不止有反正切一个应用，下面再看一个例子。  
下面是一个ball在上下运动的核心代码，重点就是ball的y轴坐标改变，就是这句：  

``` javascript
ball.y = clientY + Math.sin(angle) * range;
```

利用Math.sin(angle)的取值范围是-1到1，并且会随着angle增大而反复，使ball在一定范围上下运动。  
完整例子：[一个上下运动的球（可调参数版）][8]

``` javascript
window.onload = function () {
  const canvas = document.getElementById('canvas');
  const context = canvas.getContext('2d');
  const ball = new Ball();
  let angle = 0;
  // 运动中心
  const clientY = 200;
  // 范围
  const range = 50;
  // 速度
  const speed = 0.05;

  ball.x = canvas.width / 2;
  ball.y = canvas.height / 2;

  (function drawFrame() {
    window.requestAnimationFrame(drawFrame, canvas);
    context.clearRect(0, 0, canvas.width, canvas.height);

    ball.y = clientY + Math.sin(angle) * range;
    angle += speed;
    ball.draw(context);
  }());
};
```

### 向前运动 ###
只是上下运动不过瘾，那就让圆前进吧，其实就是每帧改变x轴的位置。  
核心代码如下，相比前面的上下运动，多了x轴的速度，每帧移动一点就形成了波浪前进的效果。  
完整实例：[一个波浪运动的球][9]

``` javascript
window.onload = function () {
  const canvas = document.getElementById('canvas');
  const context = canvas.getContext('2d');
  const ball = new Ball();
  let angle = 0;
  const centerY = 200;
  const range = 50;
  const xspeed = 1;
  const yspeed = 0.05;

  ball.x = 0;
  (function drawFrame() {
    window.requestAnimationFrame(drawFrame, canvas);
    context.clearRect(0, 0, canvas.width, canvas.height);
    ball.x += xspeed;
    ball.y = centerY + Math.sin(angle) * range;
    angle += yspeed;
    ball.draw(context);
  }());
};
```

### 其他示例 ###
其他的应用就不一一讲解，罗列出来一些：  
- [不断缩放的球][11]
- [两轴同时改变的圆][12]
- [绘制波][13]
- [一个做圆周运动的圆][14]
- [一个做椭圆形运动的圆][15]
- [计算两个随机块的距离][16]
- [中心点到鼠标的距离][17]


  [1]: https://developer.mozilla.org/zh-CN/docs/Web/API/Canvas_API
  [3]: https://nimokuri.github.io/H5Learning-animationDemo/part2/01-rotate-to-mouse.html
  [4]: https://github.com/nimokuri/H5Learning-animationDemo/blob/master/common/arrow.js
  [5]: https://github.com/nimokuri/H5Learning-animationDemo/blob/master/common/ball.js
  [6]: https://nimokuri.github.io/myBlog-backup/assets/【30分钟学完】canvas动画|游戏基础(2)：开始画画/2.png
  [8]: https://nimokuri.github.io/H5Learning-animationDemo/part2/03-bobbing-2.html
  [9]: https://nimokuri.github.io/H5Learning-animationDemo/part2/04-wave-1.html
  [11]: https://nimokuri.github.io/H5Learning-animationDemo/part2/05-pulse.html
  [12]: https://nimokuri.github.io/H5Learning-animationDemo/part2/06-random.html
  [13]: https://nimokuri.github.io/H5Learning-animationDemo/part2/07-wave-2.html
  [14]: https://nimokuri.github.io/H5Learning-animationDemo/part2/08-circle.html
  [15]: https://nimokuri.github.io/H5Learning-animationDemo/part2/09-oval.html
  [16]: https://nimokuri.github.io/H5Learning-animationDemo/part2/10-distance.html
  [17]: https://nimokuri.github.io/H5Learning-animationDemo/part2/11-mouse-distance.html
