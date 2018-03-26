## 前言

本系列前几篇中常出现物体跑到画布外的情况，本篇就是为了解决这个问题。  
阅读本篇前请先打好前面的基础。  
本人能力有限，欢迎牛人共同讨论，批评指正。  

## 越界检测

越界是常见的场景，我们以画布边界为例进行讨论，示例中矩形边界即是：  

```javascript
let top = 0;
let bottom = canvas.height;
let left = 0;
let right = canvas.width;
```
![边界][1]

假定物体是个圆形，如图其圆心坐标即是物体的x轴和y轴坐标，则可得越界条件，以下任何一项为true即可判定越界。  

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

## 越界检测

[1]: https://nimokuri.github.io/myBlog-backup/assets/【30分钟学完】canvas动画|游戏基础(4)：边界与碰撞/1.png
