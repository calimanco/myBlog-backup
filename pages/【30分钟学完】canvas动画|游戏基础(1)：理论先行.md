## 前言

本文虽说是基础教程，但这是相对动画/游戏领域来说，在前端领域算是中级教程了，不适合前端小白或萌新。阅读前请确保自己对前端三大件（JavaScript+CSS+HTML）的基础已经十分熟悉，而且有高中水平的数学和物理知识。demo采用ES6编写，遵循Airbnb规范，不依赖第三方框架或库，请在现代浏览器里运行。  
大部分例子来自《Foundation HTML5 Animation with JavaScript》，感谢这本书作者的辛劳和启发。本教程也可以算是该书的精(tian)简(you)优(jia)化(cu)版，既是我的个人读书笔记，也是经验总结，方便没空看书的忙人阅读。  
本人能力有限，欢迎牛人共同讨论，批评指正。  

## 何为动画/游戏

> 【科普】动画是指由许多帧静止的画面，以一定的速度（如每秒16张）连续播放时，肉眼因视觉残象产生错觉，而误以为画面活动的作品。为了得到活动的画面，每个画面之间都会有细微的改变。而画面的制作方式，最常见的是手绘在纸张或赛璐珞片上，其它的方式还包含了运用黏土、模型、纸偶、沙画等。

使用H5技术实现动画原理跟传统动画是一样的，都是利用“视觉暂留”现象，计算机通过一定的规则运算得到一个画面（像素数据），然后以一定速度连续播放就形成动画。但也有些许不同，传统动画的重点是绘画技法的表现，也就是每张图画得漂亮，而计算机动画更关心的是如何确立运算规则，这也是数学和物理知识的运用。  
在计算机领域，动画和游戏界限并不明显，他们的差别就是是否有交互性，如果玩家有一定的改变动画的操作，再加上一些游戏规则，那就可以称得上是游戏了。  

> 【PS】顺便一提，常有疑问为什么电脑玩游戏卡，看电影不卡呢？  
> 因为所谓的数字版电影，不论是三维动画还是二维动画，都是已渲染好的画面，也就是数据是一帧帧的图片，而计算机只需要按顺序换图片就能播放。但游戏的画面是实时计算出来的，如果计算机性能不行，也就是无法在下一帧完成渲染，画面自然就会卡顿。在PC和主机性能低下的年代，将部分游戏画面预渲染后放入游戏也是常见的提高性能的做法。

## H5相关技术概述

### canvas

> 【科普】`<canvas>` 是 HTML5 新增的元素，可用于通过使用JavaScript中的脚本来绘制图形。例如，它可以用于绘制图形，制作照片，创建动画，甚至可以进行实时视频处理或渲染。

作为上世代flash的升级替代品，简单来说就是浏览器提供一个画布，你的工作就是用js操作画笔在上面画画，不断重复画画和擦除的工作，就可以实现动画。  
[canvas相关文档][1]

### 用户交互

交互是游戏的根本，H5上的交互不外乎鼠标、触摸和键盘这几种，其实就是DOM标准的事件流。我们在事件中拿到屏幕上的坐标或键盘的键位代号，执行相应的操作。这不是本教程重点，不懂的右转HTML相关基础，这里就不细说了。  
这里有些简单例子，可以在控制台看到效果：

-   [演示所有鼠标事件][2]
-   [演示鼠标按下坐标获取][3]
-   [演示触摸事件][4]
-   [演示键盘事件][5]
-   [演示键盘上、下、左、右][6]

demo中为了方便使用封装了这些交互方法，放在工具库[utils.js][7]里，这里以获取鼠标事件，触摸事件同理为例。

```javascript
utils.captureMouse = function captureMouse(element) {
  const mouse = {
    x: 0,
    y: 0,
  };
  element.addEventListener('mousemove', (event) => {
    let x;
    let y;
    if (event.pageX || event.pageY) {
      x = event.pageX;
      y = event.pageY;
    } else {
      x = event.clientX + document.body.scrollLeft + document.documentElement.scrollLeft;
      y = event.clientY + document.body.scrollTop + document.documentElement.scrollTop;
    }
    x -= element.offsetLeft;
    y -= element.offsetTop;

    mouse.x = x;
    mouse.y = y;
  }, false);

  return mouse;
};
```

## 动画循环

这是计算机动画在代码层面的核心，简单来说就是一个循环调用（不断递归）的过程，每次循环都是一帧画面的产生，实现方式大体可以归纳为三类。

### 基于帧的动画（requestAnimationFrame）

> 【科普】在视频领域，电影、电视、数字视频等可视为随时间连续变换的许多张画面，而帧是指每一张画面。

一般来说1秒15帧就可让人眼不发觉黑暗的间隔，25帧就可感觉流畅，每秒钟帧数 (fps) 愈多，所显示的动作就会愈流畅。W3C所建议的刷新率是1秒60帧，大部分浏览器是遵循这一标准的。  
requestAnimationFrame是H5加入的函数，用法类似于setTimeout，用于告诉浏览器下一帧的时候该干什么，比如一个物体每1帧要移动多少距离。它提供**基于浏览器的优化实现**，是实现H5动画的首选，所有demo都有使用。至于它优化了什么，下面会提到。  
[requestAnimationFrame文档链接][8]

### 基于定时器的动画

在requestAnimationFrame还未出现的时代，一般是用setTimeout和setInterval实现动画，也可以使用他们模拟requestAnimationFrame，只需把时间间隔设为1000/60毫秒，也就是大约16.7毫秒执行下一个循环，当然你也可以定义自己需要的帧率，达到游戏中常见的**锁帧**效果。  
所谓模拟，另一层意思就是不可能相同，我们的程序在浏览器沙盒中运行是不知道显卡和显示器硬件实际是否在刷新的，但浏览器是可以知道的，所以浏览器才可以真正的知道什么时候屏幕会刷新，更好的配合硬件工作，这也是requestAnimationFrame优于定时器的原因。  
基于此我们可以创造一个polyfill放到工具库[utils.js][9]里。  

```javascript
if (!window.requestAnimationFrame) {
  window.requestAnimationFrame = (window.webkitRequestAnimationFrame ||
    window.mozRequestAnimationFrame ||
    window.oRequestAnimationFrame ||
    window.msRequestAnimationFrame ||
    function timeout(callback) {
      return window.setTimeout(callback, 1000 / 60);
    });
}
```

特别注意：**js中给定时器规定的时间间隔仅仅表示最少的时间，而非确切的时间**，对于过复杂需要超过时间间隔才执行完的程序，执行时间就会被延后。  

### 基于时间的动画

其实无论是requestAnimationFrame还是定时器，都不能保证以特定速率播放。也就是说复杂的动画在性能较差的计算机上播放，会比它设计速度慢。这个在游戏的体验上是十分不友好的，所以就有了游戏里常见的**跳帧**做法。  
其实就是使用真实的时间来度量每个物体的运动变化，而不是依靠每帧的变化。将物体每帧移动距离，转变为物体每秒移动距离。  
由于demo都是简单动画，所以暂时不会使用这个操作。  

## 计算机中一些数学概念与标准的差异

### 弧度(radian)与角度(degree)

日常我们使用角度会比较多，应该没有人不知道一个圆是360度吧（笑）。但计算机中不使用角度概念而是使用弧度。学校也有说过，这里就不讲解他们的关系了，只要明确一圆周是2π弧度，两者的转换用代码表示就是：

```javascript
let radians = degrees * Math.PI / 180;
let degrees = radians * 180 / Math.PI;
```

### 坐标系

计算机里的坐标系也不是日常使用的标准坐标系，可以说是标准坐标系的颠倒版本，如下图，越往右x轴值越大，越往下y轴值越大，反之亦然。  

> 【科普】这个坐标系有一定的历史背景，因为“大屁股”显示器里的电子枪是从左往右，从上往下扫描屏幕的。

![坐标系][10]

这个坐标系还导致了另一个问题，就是**角度的正负值是与标准坐标系相反的**，如下图，顺时针角度才是正值，逆时针为负值。  

![角度][11]

[1]: https://developer.mozilla.org/zh-CN/docs/Web/API/Canvas_API

[2]: https://nimokuri.github.io/H5Learning-animationDemo/part1/03-mouse-events.html

[3]: https://nimokuri.github.io/H5Learning-animationDemo/part1/04-mouse-position.html

[4]: https://nimokuri.github.io/H5Learning-animationDemo/part1/05-touch-events.html

[5]: https://nimokuri.github.io/H5Learning-animationDemo/part1/06-keyboard-events.html

[6]: https://nimokuri.github.io/H5Learning-animationDemo/part1/07-key-codes.html

[7]: https://github.com/nimokuri/H5Learning-animationDemo/blob/master/common/utils.js

[8]: https://developer.mozilla.org/zh-CN/docs/Web/API/Window/requestAnimationFrame

[9]: https://github.com/nimokuri/H5Learning-animationDemo/blob/master/common/utils.js

[10]: https://nimokuri.github.io/myBlog-backup/assets/【30分钟学完】canvas动画|游戏基础(1)：理论先行/1.png

[11]: https://nimokuri.github.io/myBlog-backup/assets/【30分钟学完】canvas动画|游戏基础(1)：理论先行/2.png
