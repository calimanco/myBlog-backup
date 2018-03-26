## 前言

本文是接续系列教程的extra1，主要是介绍颜色系统在canvas中的应用。  
本来是与extra1一起成文的，因为segmentfault莫名其妙的字数限制bug只能分割放送了。  

## canvas操纵像素

你如果认为canvas只是画图工具，那接下来的操作会颠覆你的认知。canvas提供api可以获取画布上任何一个像素，并可以自由的操作他们。  

### 获取像素

直接访问像素的功能由canvas上下文中的ImageData对象提供，它提供了以下一组方法，都会返回ImageData对象。  

-   getImageData()接受x轴坐标、y轴坐标、宽度、高度四个参数，获取画布上这个矩形区域的像素数据；  
-   createImageData()可凭空创建指定宽高的矩形区域，初始是黑色，也可以输入一个ImageData对象用于创建一个同样大小的区域，但注意**不会复制像素数据**。  

```javascript
context.getImageData(x, y, width, height);
context.createImageData(width, height);
context.createImageData(anothorImageData);
```

获取到的ImageData对象中data属性是一个一维数组，乍看乱糟糟的，但细看你会发现其实这就是RGBA的颜色数据，也就是数组中每个四位就是一个像素的颜色数据，这里注意一下**透明度A也是0~255，不是CSS里简化过的0~1**。

* * *

**举个例子**
现在假定在一个纯红色区域取一块`2*2`的矩形，我们得到的像素数据是：

```javascript
let pixels = [255, 0, 0, 255, 255, 0, 0, 255,
              255, 0, 0, 255, 255, 0, 0, 255]
```

他们与图像的对应关系是**从左到右，从上到下**，大概就像上面代码格式化这样，如图所示：  

![像素对应位置][6]

根据4对1的对应关系，我们很容易就能写出遍历的办法，offset就相当于指针，每次移动4位，代码如下：  

```javascript
for (let offset = 0, len = pixels.length; offset < len; offset += 4) {
  r = pixels[offset];
  g = pixels[offset + 1];
  b = pixels[offset + 2];
  a = pixels[offset + 3];
}
```

当需要访问特定坐标的像素时，可以使用如下公式，其中xpos是像素点在该区域的x坐标；ypos是像素点在该区域的y坐标，imagedata.width是指区域横向有多少像素。  

```javascript
let offset = (xpos + ypos * imagedata.width) * 4;
let r = pixels[offset];
let g = pixels[offset + 1];
let b = pixels[offset + 2];
let a = pixels[offset + 3];
```

### 绘制像素

可以将修改过的ImageData对象重新用上下文的putImageData()方法绘制到指定区域，该方法接受三个参数：ImageData对象、x轴坐标、y轴坐标。绘制指定的位置绘制ImageData对象的内容。直接看下面例子的核心代码。  
先绘制铺满画布的色块，点击按钮触发change事件处理器可改变颜色，过程见注释。  
完整例子：[演示反色变化][1]

> 【PS】对js了解不深的朋友可能会有疑问，遍历过程操作的是pixels，imageData怎么会改变呢？  
> 这是因为js中对象都是地址传递的特点，也就是pixels = imageData.data操作只是将pixels变量的指向到imageData.data所指向的内存空间，所以操作pixels就是操作imageData.data。

```javascript
window.onload = function () {
  const canvas = document.getElementById('canvas');
  const context = canvas.getContext('2d');
  // 绘制色块，每个色块宽10像素，高等于画布高，铺满画布
  for (let i = 0; i < canvas.width; i += 10) {
    context.fillStyle = (i % 20 === 0) ? '#f00' : ((i % 30 === 0) ? '#0f0' : '#00f');
    context.fillRect(i, 0, 10, canvas.height);
  }
};

function change() {
  const canvas = document.getElementById('canvas');
  const context = canvas.getContext('2d');
  // 获取整个画布的ImageData对象
  const imageData = context.getImageData(0, 0, canvas.width, canvas.height);
  // 取出颜色数据
  const pixels = imageData.data;
  // 遍历颜色数据求每个颜色的反色
  for (let offset = 0, len = pixels.length; offset < len; offset += 4) {
    pixels[offset] = 255 - pixels[offset];
    pixels[offset + 1] = 255 - pixels[offset + 1];
    pixels[offset + 2] = 255 - pixels[offset + 2];
    // 这里没有操作透明度
  }
  // 将ImageData重新绘制到画布上
  context.putImageData(imageData, 0, 0);
}
```

### 更多有趣的例子

canvas强大的像素操作可以给我们带来更多的可能，也许你会想开始做一个网页版的美图工具了吧（笑）。  
这里还有一些有趣的demo可以玩玩：

-   [演示灰度变化][2]
-   [有趣的色彩波纹][3]

## 综合案例

有关颜色的番外部分到这里就基本完结了，最后有个综合题，会应用这些技术。  
将系列第二篇中的[鼠标画图工具][4]改造成鼠标喷漆工具，这里建议自己动手实践一下。  
下面例子基本思路就是取得画布像素数据，每当鼠标点下并移动（执行onMouseMove）就随机改变鼠标周围一定范围的像素点的颜色。  
完整案例：[鼠标喷漆工具][5]

```javascript
window.onload = function () {
  const canvas = document.getElementById('canvas');
  const context = canvas.getContext('2d');
  // 获得整个画布区域的ImageData对象
  const imageData = context.getImageData(0, 0, canvas.width, canvas.height);
  // 取出像素数据
  const pixels = imageData.data;
  // 设定笔刷大小
  const brush_size = 25;
  // 设定笔刷密度
  const brush_density = 80;
  // 笔刷的颜色变量
  let brush_color;

  function onMouseMove() {
    // 根据设定的笔刷密度生成随机像素点
    for (let i = 0; i < brush_density; i++) {
      // 随机像素点角度相对于鼠标的角度
      const angle = Math.random() * Math.PI * 2;
      // 根据设定的笔刷大小，随机像素点以鼠标为圆心的半径
      const radius = Math.random() * brush_size;
      // 计算出像素点的x轴相对坐标
      const xpos = (mouse.x + Math.cos(angle) * radius) | 0;
      // 计算出像素点的y轴相对坐标
      const ypos = (mouse.y + Math.sin(angle) * radius) | 0;
      // 算出该像素点在pixels中的偏移量
      const offset = (xpos + ypos * imageData.width) * 4;
      // 对这个像素点的颜色数据进行操作，将颜色分解成三基色
      pixels[offset] = brush_color >> 16 & 0xff;
      pixels[offset + 1] = brush_color >> 8 & 0xff;
      pixels[offset + 2] = brush_color & 0xff;
      pixels[offset + 3] = 255;
    }
    // 重新绘制区域
    context.putImageData(imageData, 0, 0);
  }
  canvas.addEventListener('mousedown', () => {
    // 随机一个颜色
    brush_color = utils.parseColor(Math.random() * 0xffffff, true);
    canvas.addEventListener('mousemove', onMouseMove, false);
  }, false);
  canvas.addEventListener('mouseup', () => {
    canvas.removeEventListener('mousemove', onMouseMove, false);
  }, false);
};
```

[1]: https://nimokuri.github.io/H5Learning-animationDemo/part3/13-invert-color.html

[2]: https://nimokuri.github.io/H5Learning-animationDemo/part3/14-grayscale.html

[3]: https://nimokuri.github.io/H5Learning-animationDemo/part3/15-pixel-move.html

[4]: https://nimokuri.github.io/H5Learning-animationDemo/part3/01-drawing-app.html

[5]: https://nimokuri.github.io/H5Learning-animationDemo/part3/16-spray-paint.html

[6]: https://nimokuri.github.io/myBlog-backup/assets/【30分钟学完】canvas动画|游戏基础(extra1-1)：美图我也行/1.png
