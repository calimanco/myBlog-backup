## 前言

本篇主要讲坐标旋转及余弦定理的应用，这是编程动画必不可少的技术。  
阅读本篇前请先打好前面的基础。  
本人能力有限，欢迎牛人共同讨论，批评指正。  

## 坐标旋转

模拟场景：已知一个中心点（centerX,centerY），旋转前物体ball（x1,y1），旋转弧度（rotation）；求旋转后物体（x2,y2）。（如下图）  

![场景图][1]

坐标旋转就是说围绕某个点旋转坐标，我们要依据旋转的角度（弧度），计算出物体旋转前后的坐标，一般有两种方法：  

### 简单坐标旋转

灵活运用前章节的三角函数知识可以很容易解决，基本思路：  

1. 计算物体初始相对于中心点的位置；
2. 使用atan2计算弧度angle；
3. 使用勾股定理计算半径radius；
4. angle+rotation后使用cos计算旋转后x轴位置，用sin计算旋转后y轴位置。

下面是示例是采用这种方法的圆周运动，其中vr为ball相对于中心点的弧度变化速度，由于旋转半径是固定的，所以没有在动画循环里每次都获取。  
完整示例：[简单坐标旋转演示][2]  

```javascript
/**
 * 简单坐标旋转演示
 * */
window.onload = function () {
  const canvas = document.getElementById('canvas');
  const context = canvas.getContext('2d');
  const ball = new Ball();
  ball.x = 300;
  ball.y = 200;
  // 弧度变化速度
  const vr = 0.05;
  // 中心点位置设定在画布中心
  const centerX = canvas.width / 2;
  const centerY = canvas.height / 2;
  // ball相对与中心点的距离
  const dx = ball.x - centerX;
  const dy = ball.y - centerY;
  // ball相对与中心点的弧度
  let angle = Math.atan2(dy, dx);
  // 旋转半径
  const radius = Math.sqrt(dx ** 2 + dy ** 2);

  (function drawFrame() {
    window.requestAnimationFrame(drawFrame, canvas);
    context.clearRect(0, 0, canvas.width, canvas.height);

    ball.x = centerX + Math.cos(angle) * radius;
    ball.y = centerY + Math.sin(angle) * radius;
    angle += vr;
    ball.draw(context);
  }());
};
```

### 使用撒

[1]: https://nimokuri.github.io/myBlog-backup/assets/【30分钟学完】canvas动画|游戏基础(6)：坐标旋转探究/1.png

[2]: https://nimokuri.github.io/H5Learning-animationDemo/part9/01-rotate-1.html
