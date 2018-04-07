## 前言

现在cross-origin resource sharing（跨域资源共享，下简称CORS）已经十分普及，算上IE8的不标准兼容（XDomainRequest），各大浏览器基本都已支持，当年为了前后端分离、iframe交互和第三方插件开发而头疼跨域是时代已经过去，但当年为了跨域无所不用其极的风骚操作却依然值得学习。  
本篇文章不是从实用的角度考量这些旧时代的跨域手段，而是更偏向理论的阐述，并引发都浏览器安全的思考，因为跨域实际上也是各类攻击的核心。  
本人个人能力有限，欢迎大牛一起讨论，批评指正。   

## 同源策略

1995年，同源政策由Netscape公司引入浏览器。目前，所有浏览器都实行这个安全策略。  
核心是确保不同源提供的文件（资源）之间是相互独立的。换句话说，只有当不同的文件脚本是由相同的域、端口、HTTP协议提供时，才没有特殊的限制。特殊限制可以细分为两个方面：  

-   对象访问限制：主要体现在iframe，如果父子页面的源是不同的，那就不可以访问对方的DOM方法和属性（包括Cookie、LocalStorage和IndexDB等）。不同来源便抛出异常。  
-   网络访问限制：主要体现在AJAX请求，如果发起的请求目标源与当前页面不同，浏览器就会限制了发起跨站请求，或拦截返回的请求。  

**一个表格看懂什么是同源？**

| origin(URL)               | result  | reason         |
| ------------------------- | ------- | -------------- |
| `http://example.com`      | success | 协议，域名和端口号80均相同 |
| `http://example.com:8080` | fail    | 端口不同           |
| `https://example.com`     | fail    | 协议不同           |
| `http://sub.example.com`  | fail    | 域名不同           |

**至于为什么说这是个安全策略？**  
这个就要提到cookie-session机制，众所周知HTTP是无状态协议，而服务器如何知晓用户的登录状态？传统上是使用了cookie-session这一机制，也就是服务器为每个访问者生成了一个session标识，而session标识会被服务器包含在应答头中返回，浏览器解析到应答头中的set-cookie就把这串session标识保存到本地cookie中，利用cookie每次请求同一个域都会带上的特性，服务器器就能知晓当前的用户登录状态。  
所以如果让浏览器向不同源发起请求，就会造成很大的危险。比如用户登录了银行的网站A，也就是说A站已经在浏览器留下了cookie，这时候用户又访问了B站，如果能在B站页面上发起A站的请求，就相当于B站可以冒充用户，在A站为所欲为。  
由此可见，"同源政策"是必需的，否则cookie可以共享，互联网就毫无安全可言了。  

## 跨域方案

同源策略提出的时代还是传统MVC架构（jsp，asp）盛行的年代，那时候的页面靠服务器渲染完成了大部分填充，内容也比较简单，开发者也不会维护独立的API工程，所以其实跨域的需求是比较少的。  
新时代前后端的分离和第三方JSSDK的兴起，我们才开始发现这个策略虽然大大提高了浏览器的安全性，但有时很不方便，合理的用途也受到影响。比如：

1.  独立的API工程部署为了方便管理使用了独立的域名；
2.  前端开发者本地调试需要使用远程的API；
3.  第三方开发的JSSDK需要嵌入到别人的页面中使用；
4.  公共平台的开放API。

于是乎，在没有标准规范的时代，如何解决这些问题的跨域方案就被纷纷提出，可谓百家争鸣，其中不乏令人惊叹的骚操作，这样的极客精神依然值得我们敬佩和学习。  

### JSON-P

JSON-P是各类跨域方案中流行度较高的一个，现在在某些要兼容旧浏览器的环境下还会被使用，著名的jQuery也封装其方法。请勿见名知义，名字中的P是padding“带填充”的意思，这个方法在通信过程中使用的并不是普通的json，而是**自带填充功能的JavaScript脚本**。  
如何理解“自带填充功能的JavaScript脚本”，看看下面的例子或许比较简单，如果一个js文件里这样写并被引入，则全局下就会有data对象，也就是说**利用js脚本的引入和解析可以用来传递数据**，如果把js脚本换成函数运行命令岂不是可以调用全局函数了。这就是JSON-P方法的核心思想，它填充的是全局函数的数据。  

```javascript
var data = {
  a: 1,
  b: 2
}
```

> 【PS】`<script>`标签不受同源策略限制，但只能发起get请求。

**原理及流程**

1.  先定义好回调函数，也就是引入的js脚本中要调用的函数；
2.  新建`<script>`标签，将标签插入页面浏览器便会发起get请求；
3.  服务器根据请求返回js脚本，其中调用了回调函数。

![jsonp流程图][1]

```javascript
// 定义回调函数
function getTheAnimal(data){
	var myAnimal = data.animal;
}
// 新建标签
var script = document.createElement("script");
script.type = "text/javascript";
// 常用的在url参数部分跟服务器约定号回调函数名
script.src = "http://demo.com/animal.json?callback=getTheAnimal";
document.getElementByTagName('head')[0].appendChild(script);
```

**总结**  

优点：  

-   简单，有现成的工具库（jQuery）支持；
-   支持上古级别的浏览器（IE8-）。

缺点：

-   只能是GET方法；
-   受浏览器URL最大长度2083字符限制；
-   无法调试，服务器错误无法检测到具体原因；
-   有CSRF的安全风险；
-   只能是异步，无法同步阻塞；
-   需要特殊接口支持，不能基于REST的API规范。

### 子域名代理

这个方法实际上是利用浏览器允许iframe内的页面只要是跟父页面是同个一级域名下，就能被父页面修改和调用的特点。也许你会疑问，上面讲同源策略的表格中很明确二级域名不同也是算不同源，这岂不矛盾了？  
这其实不矛盾，如果正常操作确实会被同源策略限制，但浏览器的`document.domain`**允许网站将主机部分更改为原始值的后缀**。这意味着，寄放在sub.example.com的页面可以将它的源设置为example.com，但是并不能将其设置为alt.example.com或google.com。  

> 【PS】这里有一个细节，父子页面均要设置`document.domain`才能被互相访问，单一一个是无法跨域的。`document.domain`的特点：只能设置一次；只能更改域名部分，不能修改端口号和协议；重置源的端口为协议默认端口。

**原理及流程**

1.  新建一个子域，比如api.demo.com（页面在主域名demo.com下）；  
2.  子域下需要一个代理文件proxy.html，设置其`document.domain = 'demo.com'`，并可以包含发起ajax的工具；
3.  所有API地址都是在api.demo.com；
4.  把需要发请求的主域页面设置其`document.domain = 'demo.com'`；
5.  新建iframe标签链接到代理页；
6.  当iframe内的子页面就绪时，父页面就可以使用子页面发起ajax请求。

![子域名代理流程图][2]

```html
// 最简单的代理文件proxy.html
<!DOCTYPE html>
<html>
	<script>
		document.domain = 'demo.com';
	</script>
	<script src="jquery.min.js"></script>
</html>
```

```javascript
// 新建iframe
var iframe = document.createElement('iframe');
// 链接到代理页
iframe.src = 'http://api.demo.com/proxy.html';
// 代理页就绪时触发
iframe.onload = function(){
  // 由于代理页已经和父页设置了相同的源，父的脚本可以调用代理页的ajax工具；
  // 由于是在子页面发起，其请求地址就跟子页面同源了。
  iframe.contentWindow.jQuery.ajax({
    method: 'POST',
    url: 'http://api.demo.com/products',
    data: {
      product: id,
    },
    success: function(){
      document.body.removeChild(iframe);
      /*...*/
    }
  })
}
document.getElementsByTagName('head')[0].appendChild(iframe);
```

**总结**  

优点：  

-   可以发送任意类型的请求；
-   可以使用基于REST的API规范。

缺点：  

-   不太适合第三方API，给第二方使用较麻烦；
-   iframe对浏览器性能影响较大；
-   无法使用非协议默认端口的API。

### 模拟form表单

form表单的target属性可以指定一个iframe，使主页面不跳转，而iframe内跳转，所以这个方法的核心就是**利用表单提交，并在iframe中获取数据**。  
要访问iframe内外页面互访也是必须设置同源，这点与子域代理是相似的；而iframe内回调父页面，又与JSON-P相似，可以说是两个思路的合体版。  
form表单提交后返回的是页面，所以与JSON-P不同的是，返回的是**包含了自带填充功能的JavaScript脚本的页面**，说起来有点绕，简单来说就是把JSON-P返回的脚本放到一个html页面里自运行。  
相比子域代理的方法，它**不需要代理页**。  

> 【PS】form表单提交的特点就是会导致整个页面跳转，返回数据是在新的页面上，这样自然不会产生跨域的问题。  

**原理及流程**

1.  新建一个子域，比如api.demo.com（页面在主域名demo.com下）；  
2.  所有API地址都是在api.demo.com；
3.  把需要发请求的主域页面设置其`document.domain = 'demo.com'`；
4.  先定义好父页面上的回调函数；
5.  新建iframe标签并指定名字；
6.  新建表单form标签，指定target为刚才的iframe，并添加数据；
7.  提交表单，iframe内跳转，其中自运行脚本调用了父页面的回调函数。

![模拟form表单流程图][3]

```javascript
// 新建并隐藏iframe
var frame = document.createElement('iframe');
iframe.name = 'post-review';
frame.style.display = 'none';

// 新建表单
var form = document.createElement('form');
form.action = 'http://api.demo.com/products';
form.method = 'POST';
form.target = 'post-review';
// 添加数据
var score = document.createElement('input');
score.name = 'score';
score.value = '5';
// 添加数据
var message = document.createElement('input');
message.name = 'message';
message.value = 'hello world';
// 把数据加到表单
form.appendChild(score);
form.appendChild(message);
// 渲染iframe和表单
document.body.appendChild(frame);
document.body.appendChild(form);
// 提交表单发起请求
form.submit();
// 完成清理元素
document.body.removeChild(form);
document.body.removeChild(frame);
```

```html
// 最简单返回html
<!DOCTYPE html>
<html>
	<script>
		document.domain = 'demo.com';
		window.parent.jsonpCallback('{"status":"success"}');
	</script>
</html>
```

**总结**  

由于这个方法是JSON-P与子域名代理的结合版，可以说即拥有两者的优点，也保留了两者一些缺点。  

优点：  

-   可以发送任意类型的请求；
-   不需要代理页；
-   支持上古级别的浏览器（IE8-）。

缺点：  

-   不太适合第三方API，给第二方使用较麻烦；
-   iframe对浏览器性能影响较大；
-   无法使用非协议默认端口的API；
-   需要特殊接口支持，不能基于REST的API规范。

### window.name

这方法利用了`window.name`的特性：一旦被赋值后，当窗口被重定向到一个新的URL时不会改变它的值。这一行为使得不同域的特定文档可以读取该属性值，因此可以绕过同源策略并使跨域消息通信成为可能。  

> 【PS】例子里演示的是发起get请求，只要把请求地址直接写到src里就行了。如果想要发起其他类型的请求，可以类比采用模拟的form的方式进行改造。

**原理及流程**

1.  新建iframe，使用iframe访问一个非同源的地址（发请求）；
2.  当页面加载完成后，iframe内脚本给window.name属性赋值，这时父页面还是不能读取到子页面的属性（因为不同源）；
3.  iframe自身回调到一个同源的地址（可能只是个空白页），这时候window.name没有改变；
4.  父页面顺利读取window.name的值。

![window.name流程图][4]

```javascript
// 新建iframe
var iframe = document.createElement('iframe');
var body = document.getElementByTagName('body');
// 隐藏iframe并链接地址
iframe.style.display = 'none';
iframe.src = 'http://api.demo.com/server.html?id=1';
// 因为需要两次跳转，这里有个完成标记
var done = fasle;
// 这里会触发至少两次，一次由于非同源是取不到值的。
iframe.onload = iframe.onreadystatechange = function(){
	if(! this.readyState && (iframe.readyState !== 'complete' || done)){
		return;
	}
	console.log('Listening');
	var name = iframe.contentWindow.name;
	if(name){
		console.log(iframe.contentWindow.name);
		done = true;
	}
};
body.appendChild(iframe);
```

```html
// 最简单返回html
<!DOCTYPE html>
<html>
	<script>
	function init(){
		window.name = 'hello';
		window.location = 'http://demo.com/empty.html'
	}
	</script>
	<body onload="init();"></body>
</html>
```

**总结**  

优点：  

-   可以发送任意类型的请求；
-   不需要设置子域名。

缺点：  

-   iframe对浏览器性能影响较大；
-   需要特殊接口支持，不能基于REST的API规范；
-   每当你想要获取一条新的消息时都不得不发起两次网络请求，网络成本大；
-   需要准备空白页，对它的访问是无意义的，影响流量统计。

### window.hash

这个方法利用了location的特性：不同域的页面，可以写不可读。而只改变哈希部分（井号后面）不会导致页面跳转。也就是可以让父、子页面互相写对方的location的哈希部分，进行通讯。

**原理及流程**

1.  新建iframe，使用iframe访问一个非同源的地址（发请求），参数里带上父页面url；
2.  当页面加载完成后，iframe内脚本设置父页面的url并在哈希部分带上数据；
3.  父页面的脚本循环检查哈希值的变化，如果检查到有值就取值并清空哈希值；

> 【PS】父页面会循环检查哈希是否改变来读取值，因为这种降级方案的使用环境一般是不会有hashchange事件的。演示里是最简单的get方法，如果想要发起其他类型的请求，可以类比采用模拟的form的方式进行改造，但记住不要丢失父页面的url。

![window.hash流程图][5]

```javascript
// 获取当前url
var url = window.location.href;
// 新建iframe
var iframe = document.createElement('iframe');
// 隐藏iframe并设置链接，把当前url带上
iframe.style.display = 'none';
iframe.src = 'http://api.demo.com/server.html?id=1&url=' + encodeURIComponent(url);

var body = document.getElementByTagName('body')[0];
body.appendChild(iframe);
// 循环监听处理
var listener = function(){
	// 读取
	var hash = location.hash;
	// 还原
	if(hash && hash !== '#'){
		console.log(hash.replace('#', ''));
		window.loacation.href = url + '#';
	}
	// 继续监听
	setTimeout(listener, 100);
};
listener();
```

```html
// 最简单返回html
<!DOCTYPE html>
<html>
	<script>
	function init(){
		// 剪裁出父页面的url
		var parentUrl = '';
		var url = window.location.href;
		var str = url.split('?')[1].replace('?', '');
		strs = str.split("&");
		for(var i = 0; i < strs.length; i ++) {
			if(strs.split("=")[0] === 'url'){
				parentUrl = strs.split("=")[1];
			}
		}
		// 设置到父页面上
		window.parent.location = decodeURIComponent(parentUrl) + '#helloworld';
	}
	</script>
	<body onload="init();"></body>
</html>
```

**总结**  

优点：  

-   可以发送任意类型的请求；
-   不需要设置子域名。

缺点：  

-   iframe对浏览器性能影响较大；
-   需要特殊接口支持，不能基于REST的API规范；
-   循环检查哈希需要消耗性能；
-   返回数据受浏览器URL最大长度2083字符限制。

## 现代的标准

W3C的标准化跨域方案，让现代浏览器跨域已经不是什么复杂的事。这部分网上资料已经很多，这里就只是简单介绍。  

### CORS

CORS是一个W3C标准，全称是"跨域资源共享"（Cross-origin resource sharing）。  
它允许浏览器向跨源服务器，发出XMLHttpRequest请求，从而克服了AJAX只能同源使用的限制。  

[CORS参考文档][6]  
[跨域资源共享 CORS 详解][7]

### postMessage

H5的window.postMessage为浏览器带来了一个安全的。基于事件的消息api。  
只要是window对象，基本都可以使用这个方法，也就是说window.name、window.hash这类风骚的操作都已成为降级方案。  

[postMessage参考文档][8]  

## 安全问题

上述的各类非标准的骚操作，都算是对同源策略的破解办法，在方便开发者完成跨域目的的同时，各类恶意的攻击者也自然会利用这些方案为非作歹。  
其中子域名代理的风险最低，因为需要服务器设置特定的子域名，也就是已经是两个源的协商结果，一般黑客是难以模拟的。  
风险最高的要算JSON-P的方案，因为这是任何客户端都可随意使用的办法，CSRF攻击的核心也是利用了特定标签的跨域性发起请求，所以JSON-P最好用在无用户状态的低安全性API上。  

[1]: https://nimokuri.github.io/myBlog-backup/assets/【Geek议题】当年那些风骚的跨域操作/1.png

[2]: https://nimokuri.github.io/myBlog-backup/assets/【Geek议题】当年那些风骚的跨域操作/2.png

[3]: https://nimokuri.github.io/myBlog-backup/assets/【Geek议题】当年那些风骚的跨域操作/3.png

[4]: https://nimokuri.github.io/myBlog-backup/assets/【Geek议题】当年那些风骚的跨域操作/4.png

[5]: https://nimokuri.github.io/myBlog-backup/assets/【Geek议题】当年那些风骚的跨域操作/5.png

[6]: https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS

[7]: http://www.ruanyifeng.com/blog/2016/04/cors.html

[8]: https://developer.mozilla.org/zh-CN/docs/Web/API/Window/postMessage
