## 前言

web前端发展到现代，已经不再是严格意义上的后端MVC的V层，它越来越向类似客户端开发的方向发展，已独立拥有了自己的MVVM设计模型。前后端的分离也使前端人员拥有更大的自由，可以独立设计客户端部分的架构。  

> 【科普】MVVM是Model-View-ViewModel的简写。它本质上就是MVC 的改进版。MVVM 就是将其中的View 的状态和行为抽象化，让我们将视图 UI 和业务逻辑分开。当然这些事 ViewModel 已经帮我们做了，它可以取出 Model 的数据同时帮忙处理 View 中由于需要展示内容而涉及的业务逻辑。

Vue作为现在流行的MVVM框架，也是本人平常业务中用得最多的框架。如何才能更合理、优雅的写VueSPA，是本人一直研究的课题，经过一年左右的思考和实践总结出本文。  
本文属于中高级实践讨论，不适合新手。  
本人个人的观点，不代表是最佳实践，欢迎大牛一起讨论，批评指正。  

## 工程搭建

秉着不重复造轮子的原则（其实就是懒），工程直接使用Vue2.0官方脚手架生成，使用最新webpack模板。与标准模板的主要差异：  

1.  增加了Sass预编译器
2.  增加了Vuex状态管理
3.  增加了Axios基础Ajax工具库

新增部分的安装请参考他们各自的文档，这里不赘述。  

## 项目结构

### 模拟需求

讨论架构前我们需要一个项目需求，这里简单模拟一个。  
需求点:3个一级页面，2个二级页面，底部的tabbar只在一级页面出现，首页、个人中心和登录页面是未登录也可以进入；财务和编辑个人信息是只有登录用户可见，简单原型如下：  

![原型][1]

### 开发目录

下面不讨论脚手架生成的部分目录，只聚焦src开发目录，依据原型我们可以大致规划出下面的目录：    

    ├── build
    ├── config
    ├── dist
    ├── src  开发目录
    │   ├── api  公共api集
    │   │   ├── axiosConfig.js  axios实例配置
    |   |   └── index.js  公共api集入口
    │   ├── assets  资源目录
    │   │   ├── images  图片
    │   │   ├── scripts  第三方脚本
    |   |   └── styles  基础样式库
    │   ├── components  公共组件
    │   │   ├── common  一般通用组件
    │   │   ├── form  表单通用组件
    │   │   └── popup  弹出类通用组件
    │   │── config  项目配置
    │   │   ├── dev.env.js  开发模式配置
    │   │   ├── env.js  一般配置
    │   │   ├── modules.js  模块配置
    │   │   └── prod.env.js  生产模式配置
    │   │── mixin  用于vue文件混合的模板
    │   │── modules  模块
    │   │   ├── finance  财务模块
    │   │   │   ├── components  财务模块私有组件
    │   │   │   │   └── FinanceIndexItem.vue  财务模块首页里的条目项
    │   │   │   ├── pages  财务模块页面
    │   │   │   │   └── FinanceIndex.vue  财务模块首页
    │   │   │   ├── api.js  模块api集
    │   │   │   ├── index.js  模块入口
    │   │   │   ├── Layout.vue  模块承载页
    │   │   │   └── router.js  模块内路由
    │   │   ├── home  首页模块（子目录同上）
    │   │   └── user  用户模块（子目录同上）
    │   │── pages  公共页面
    │   │   ├── Success.vue  公共状态管理模块
    │   │   └── NotFound.vue  用户模块（子目录同上）
    │   ├── router  路由管理
    │   ├── store  公共状态管理
    │   │   ├── modules  公共状态管理模块
    │   │   │   ├── com.js  通用状态
    │   │   │   └── user.js  用户状态
    │   │   └── index.js  公共状态管理入口
    │   └── utils  基础工具
    └── static

### 一些规范约定

根据本人个人开发经验总结的规范，不代表必须这么做。  

1.  所有vue组件都以大写字面开头的驼峰命名法命名，这样保持到模板代码上，可以便于区分开html的原生标签；  
2.  人为划分vue组件为“页面”和“页面上的组件”，原则上“页面上的组件”不发请求，不改变公共状态，全部通过事件交由“页面”完成，本人更倾向用˙集中管理。（其实vue中并没有页面概念）；  
3.  各个模块，包括路由管理、公共状态管理、接口集等都在目录下有个index.js的入口文件，方便引用；  
4.  基础工具内的工具使用函数式编程，做到可移植，不要对本项目产生依赖；  
5.  资源图片只在项目中保留小图（就是会被webpack处理成base64那些），大图应使用cdn，可以动态获取也可以把地址写到一个脚本里；  
6.  使用eslint使js代码符合Airbnb规范。  

## 低耦合模块化开发

项目过程中常遇到要把原来的项目分开部署，或是组件间耦合、或是多人开发时组件冲突等问题。本人提出的解决办法是将项目细分成模块进行开发，每个模块由若干相关“页面”组成，拥有私有组件、路由、api等，如示例所示：划分了三个模块，首页模块、财务模块、用户模块。  

> 【小结】这种方案的核心就是要将太过零散的组件（页面）聚合成模块，每个模块都有一定迁移性，互不耦合，实现按需打包，并且在代码分割上比单纯的分页面加载更加灵活可控。  

### Layout模块承载页

这个是为了让开发这个模块的程序员有类似根组件`<App>`的公共空间。从路由的角度来说，所有的模块内页面都是它的子路由，这样隔离了对全局路由的影响，至少路径定义可以随意些。  
一般来说它只是个空的路由跳转页，当然你把模块的公共数据放这里也可以的，在子路由就能`this.$parent`拿到数据，可以当成子路由间的bus使用，如下以示例的user模块为例：  

```html
<template>
  <router-view/>
</template>
<script>
export default {
  name: 'user',
  data(){
    return {
      name: '大白',
      age: 12,
    };
  },
};
</script>
```

### 模块内路由

模块内路由最后都会被导入总路由中，不要以为只是简单合并了文件，这里的设计也跟Layout模块承载页有关，
下面以user模块为例，我们把个人中心、登录和修改个人信息这三个页面归为user模块，路由规划如下。  

-   个人中心：`/user`
-   登录：`/user/login`
-   修改个人信息：`/user/userInfo`

其中由于“个人中心”是一级页面，需求要求底部有tabBar，所以使它只能是一级路由。  
接下来你会发现Layout模块承载页的路由路劲也是'/user'，这里不用担心会乱，因为路由管理是按顺序匹配的，至于为什么要路径一样，这只是为了满足路由规划，让路径好看而已。  

```javascript
// 通用的tabbar
import IndexTabBar from '@/components/common/IndexTabBar';
// 模块内的页面
import UserIndex from './pages/UserIndex';
import UserLogin from './pages/UserLogin';
import UserInfo from './pages/UserInfo';

export default [
  // 一级路由
  {
    name: 'userIndex',
    path: '/user',
    meta: {
      title: '个人中心',
    },
    components: {
      default: UserIndex,
      footer: IndexTabBar,
    },
  },
  {
    path: '/user',
    // 这里分割子路由
    component: () => import('./layout.vue'),  
    children: [
      // 二级路由
      {
        name: 'userLogin',
        path: 'login',
        meta: {
          title: '登录',
        },
        component: UserLogin,
      },
      {
        name: 'userInfo',
        path: 'info',
        meta: {
          title: '修改个人信息',
          requiresAuth: true,
        },
        component: UserInfo,
      },
    ],
  },
];
```

模块承载页以懒加载的形式`component: () => import('./layout.vue')`引入，这会使webpack在此处分割代码，也就是说进入模块内是需要再此请求的，可以减少首次加载的数据量，提高速度。  
[官方关于懒加载的文档][2]  
这里你会发现后续的子路由，又是以直接引入的方式加载，也就是说整个模块会一起加载，实现了**分模块加载**。  
这与简单的分页面加载不同，分页面加载一直有个难点，就是分割的量比较难把握（太多会增加请求次数，太少又降低了速度），而分模块可以将相关页面一起加载（跟提高缓存命中率很像），可以更灵活的规划我们的加载，最终效果：

1.  用户进入应用，首页的三个页面（有tabbar的）就已经加载完毕，这时点击哪个tabbar按钮都能流畅；
2.  当用户进入某个页面内的子页面，会产生一次请求；
3.  这时整个模块的页面都加载完（不一定要全部），用户在这个模块内又能流畅访问。

### 模块api集

这个设计跟模块内路由类似，目的也是为了按需加载和隔离全局。  
下面也是以user模块的模块api集为例，可以发现和路由有一些不同就是这里为了防止模块跟全局耦合，运用函数式编程思想（类似于依赖注入），将全局的axios实例作为函数参数传入，再返回出一个包含api的对象，这个导出的对象将会被**以模块名命名**，合并到全局的api集中。  

```javascript
export default function (axios) {
  return {
    postHeadImg(token, userId, data) {
      const options = {
        method: 'post',
        name: '换头像',
        url: '/data/user/updateHeadImg',
        headers: {
          token,
          userId,
        },
        data,
      };
      return axios(options);
    },
    postProduct(token, userId, data) {
      const options = {
        method: 'post',
        name: '提交产品选择',
        url: '/product/opt',
        headers: {
          token,
          userId,
        },
        data,
      };
      return axios(options);
    },
  };
}
```

### 模块入口

为了方便引用，每个模块目录下都有一个index.js，引入模块的时候可以省略，node会自动读这个文件。  
还是以user模块为例，这里主要是引入模块专属api和模块内路由，并定义了模块的名字，这个名字是后面挂载专属api是时候用的。  

```javascript
import api from './api';
import router from './router';

export default {
  name: 'user',
  api,
  router,
};
```

### 按需打包

示例中config目录下有个modules.js文件是指定打包需要的模块，测试一下打包不同数量的模块，会发现产品文件大小会改变，这就证明了已经实现按需打包。
至于路由和api集的子模块整合实现，后面会提到。   

```javascript
import home from '@/modules/home';
import finance from '@/modules/finance';
import user from '@/modules/user';

export default [
  home,
  finance,
  user
]
```

## api集的配置

> 【背景】示例项目模拟常见的接口约定，服务器与应用交互有两个自定义头部：token和userId。token是权限标识符，几乎全部api都需要带上，为了防CSRF；userId是登录状态标识符，有些需要登录状态才能使用的接口才需要带上，这两个标识符都有有效期。本示例暂不考虑自动续期的机制。

在api管理方面本人比较喜欢集中管理接口和配置，但发起请求和请求回调倾向与每个接口单独处理。  

### 导出axios实例

axios是比较流行的ajax的promise封装。[axios官方文档][3]  
本人推荐在全局保留唯一的axios实例，所有的请求都使用这个公共实例发起，实现配置的统一。  
示例项目的在api文件夹下的axiosConfig.js就是axios的配置，主要是导出一个符合项目设置的实例，并进行一些拦截器设置。  

> 【PS】至于为什么到导出实例而不是直接修改axios默认值？  
这是为了预防某些特例情况下公共实例无法满足需求，需要单独配置axios的情况，所以为了不污染原始的axios默认值，不推荐修改默认值。

```javascript
// 引入axios包
import axios from 'axios';
// 引入环境配置
import env from '../config/env';
// 引入公共状态管理
import store from '../store/index';

// 全局默认配置
const myAxios = axios.create({
  // 跨域带cookie
  withCredentials: true,
  // 基础url
  baseURL: `${env.apiUrl}/${env.apiVersion}`,
  // 超时时间
  timeout: 12000,
});

// 请求发起前拦截器
myAxios.interceptors.request.use((_config) => {
  // ...
  return config;
}, () => {
  // 异常处理
});

// 响应拦截器
myAxios.interceptors.response.use((response) => {
  // ...
}, (error) => {
  // 异常处理
  return Promise.reject(error);
});

export default myAxios;
```

### 公共api集

项目的所有公共api都会编写到这里，实现集中化管理，最后公共api集会挂载到vue根实例下，使用`this.$api`就可以方便的访问。  
由于token和userId不是必须头部，这里我推荐每个接口函数都单独处理，按需传入，这样api函数也能更加清晰。
给每个接口起名字，是为了后续取消请求所设计的。   
整体思路：**先定义公共api，再将模块内api（按需）挂载进来**，最后导出api集。   

```javascript
// 引入已经配置好的axios实例
import axios from './axiosConfig';
// 引入模块
import modules from '../config/modules';

const apiList = {
  // 获取token不需要
  getToken() {
    const options = {
      method: 'post',
      name: '获取token',
      url: '/token/get',
    };
    return axios(options);
  },
  loginWithName(token, data) {
    const options = {
      method: 'post',
      name: '用户名密码登录',
      url: '/data/user/login4up',
      headers: {
        token,
      },
      data,
    };
    return axios(options);
  },
  postHeadImg(token, userId, data) {
    const options = {
      method: 'post',
      name: '换头像',
      url: '/data/user/updateHeadImg',
      headers: {
        token,
        userId,
      },
      data,
    };
    return axios(options);
  },
};
// 使每个模块里的api集挂载到以模块名为名的命名空间下
modules.forEach((i) => {
  Object.assign(apiList, {
    [i.name]: i.api(axios),
  });
});

export default apiList;
```

## 路由管理配置

### 导入模块内路由

使用示例中用router文件夹下的index.js配置全局路由，api集类似实现集中化管理，导出路由实例会挂载到vue根实例下，使用`this.$router`就可以方便的访问。  
配置参考官方文档，这里主要提的一点是，模块内路由的整合，见实例代码段。  

```javascript
Vue.use(Router);
// 路由配置
const routerConfig = {
  routes: [
    {
      path: '/',
      meta: {
        title: env.appName,
      },
      redirect: { name: 'home' },
    },
    {
      name: 'success',
      path: '/success',
      meta: {
        title: '成功',
      },
      component: Success,
    },
    {
      path: '*',
      component: NotFound,
    },
  ],
};
// 将模块内的路由拼接到全局
modules.forEach((i) => {
  routerConfig.routes = routerConfig.routes.concat(i.router);
});
const router = new Router(routerConfig);
```

### 在路由钩子函数中处理标题和权限

路由的钩子函数有很多妙用，这里列举了一些例子。  
路由元信息meta可以自定义需要的数据，相当于给路由一个标记，然后在router.afterEach钩子函数中可以读取到并进行处理。  
回顾上面示例的模块内路由，meta中定义了title（标题）和requiresAuth（是否要登录状态），这就会在这里体现出用处。把登录权限设置在这里判断是为了防止用户进入某些需要权限的“页面”。  

```javascript
router.beforeEach((to, from, next) => {
  // 关闭公共弹框
  if (window.loading) {
    window.loading.close();
  }
  // 设置微信分享（如果有）
  wxShare({
    title: '哇哈哈',
    desc: '在路由钩子函数中处理标题和权限',
    link: env.shareBaseUrl,
    imgUrl: env.shareBaseUrl + '/images/shareLogo.png'
  });
  // 设置标题
  document.title = to.meta.title ? to.meta.title : '示例';
  // 检查登录状态
  if (to.meta.requiresAuth) {
    // 目标路由需要登录状态
    // ...
  }
  next();
});
```

## 自动化管理权限标识符（token）

权限标识符的特点就是几乎每个链接都要带上，需要维护有效期，为了不浪费服务器资源还需要持久化并保证请求唯一。本人比较推荐使用**公共状态管理vuex进行自动化管理**，减少代码编写时的顾虑。  

### 妙用公共状态管理获取token

实例中公共状态中的com模块里有tokenObj和waitToken两个字段，其中tokenObj包含了token和过期时间，waitToken是一个标记是否当前在获取token的布尔值。  

> 【PS】为什么要token保证唯一一次请求？  
常见的场景：当用户进入应用，这时候token要么没有要么已过期，这时页面需要并发两个ajax请求，由于都没有token，不唯一化处理的话，会同时先发起两个token请求，这样首先是浪费了请求资源，其次由于是异步请求，不能保证两次token的顺序，如果服务器对token管理较严格则会出问题。  

由于获取token是异步操作，所以getToken写在actions中，把主要过程包裹成立即执行函数，并通过waitToken判断是否要等待，如果要等待就隔一段时间再检查，这样就保证了并发请求时，token能唯一。

```javascript
const actions = {
  // needToRegain是为了特殊条件下强制获取使用
  getToken({ commit, state: _state }, needToRegain) {
    return new Promise((resolve, reject) => {
      (function main() {
        // 如果waitToken为真即表示发起了请求但还未回应
        if (_state.waitToken) {
          console.log('等待token');
          setTimeout(() => {
            main();
          }, 1000);
          return;
        }
        // 是否过期标记
        let isExpire = false;
        // 提取现有的tokenObj
        let tokenObj = {
          ..._state.tokenObj,
        };
        // 如果没有token就从本地存储中读取
        if (!tokenObj.token) {
          tokenObj = JSON.parse(localStorage.getItem('tokenObj'));
          // 如果本地有tokenObj会顺便添加到状态管理
          if (tokenObj) {
            commit('setTokenObj', tokenObj);
          }
        }
        // token是否过时
        if (tokenObj && tokenObj.token) {
          isExpire = new Date().getTime() - tokenObj.expireTime > -10000;
        }
        // 综合判断是否需要获取token
        if (!tokenObj || !tokenObj.token || isExpire || needToRegain) {
          commit('setWaitToken', true);
          api.getToken().then((res) => {
            // 检查返回的数据
            const checkedData = connect.dataCheck(res);
            if (checkedData.isDataReady) {
              const newTokenObj = {
                token: checkedData.data.token,
                expireTime: new Date().getTime() + (checkedData.data.expire_time * 1000),
              };
              // 设置TokenObj会顺便保留一份到本地存储
              commit('setTokenObj', newTokenObj);
              commit('setWaitToken', false);
              console.log('获取token成功');
              resolve(newTokenObj.token);
            } else {
              commit('setWaitToken', false);
              console.error('获取token失败');
              reject(checkedData.msg);
            }
          }).catch((err) => {
            window.toast('网络错误');
            commit('setWaitToken', false);
            reject(err);
          });
        } else {
          console.log('token已存在，直接返回');
          resolve(tokenObj.token);
        }
      }());
    });
  },
};
```

### token在请求代码中使用

将需要token的api函数套在getToken的回调中，就能方便的使用，不用再担心token是否过期。  

```javascript
const sendData = {
  mobile: this.formData1.mobile,
};
this.$store.dispatch('getToken').then((token) => {
  this.$api.sendSMS(token, sendData).then((res) => {
    const checkedData = this.$connect.dataCheck(res);
    if (checkedData.isDataReady) {
      window.toast('验证码已发送，请查收短信');
    } else {
      window.toast('验证码发送失败');
    }
  }).catch(() => {
    window.toast('网络错误');
  });
});
```

[1]: https://nimokuri.github.io/myBlog-backup/assets/【Geek议题】合理的VueSPA架构讨论/1.png

[2]: https://router.vuejs.org/zh-cn/advanced/lazy-loading.html

[3]: https://github.com/axios/axios
