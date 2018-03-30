接上篇《【Geek议题】合理的VueSPA架构讨论（上）》。  

## 自动化维护登录状态

登录状态标识符跟token类似，都是需要自动维护有效期，但也有些许不同，获取过程只在用户登录或注册的时候，不需要自动获取。  
本人比较推荐使用**公共状态管理vuex进行自动化管理**，并配合路由钩子，减少代码编写时的顾虑。  

### 妙用公共状态管理维护userId

示例中公共状态管理中的user模块里定义了userIdObj，其中包含了userId登录状态标识符和过期时间。  
维护userId是否过期主要是通过vuex中的getter来实现。  

```javascript
const getters = {
  getUserId: (_state) => {
    // 获取公共状态中的userIdObj
    const userIdObj = {
      ..._state.userIdObj,
    };
    // 是否过期标识
    let isExpire = false;
    // 判断是否过期
    if (userIdObj && userIdObj.userId) {
      isExpire = new Date().getTime() - userIdObj.expireTime > -10000;
    }
    // 如果过期则返回空字符串（不一定）
    if (!userIdObj || !userIdObj.userId || isExpire) {
      return '';
    }
    // 没过期则返回userId
    return userIdObj.userId;
  },
};
```

### 路由钩子中处理userId

回顾上篇中全局的路由钩子`router.beforeEach`，当目标路由元信息requiresAuth为true则表示，这个路由必须有登录状态才能访问，这时候就会进行登录状态检查。处理思路如下：  

1.  使用上面定义的getter方法获取userId；
2.  如果能获取到则说明有**有效的userId**，则时候即可跳转到目标页；
3.  如果获取到空字符串，则说明**userId无效**或**userId不存在**，跳转至登录页面。

> 【PS】示例这里的处理还不完美，最好跳转登录前保存好目标路由，登录成功就直接跳转去该路由。  

```javascript
router.beforeEach((to, from, next) => {
  // ...
  // 检查登录状态
  if (to.meta.requiresAuth) {
    console.log('目标路由需要登录状态');
    if (!store.getters.getUserId) {
      console.log('内存无登录信息，尝试在本地存储中找');
      const localUserIdObj = JSON.parse(localStorage.getItem('userIdObj'));
      if (localUserIdObj) {
        // 如果本地存储中有userIdObj，则提交到公共状态
        store.commit('setUserIdObj', localUserIdObj);
      }
    }
    // 再次检查公共状态里有没有userId
    if (!store.getters.getUserId) {
      console.log('依旧无登录信息');
      router.push({
        name: 'userLogin',
      });
    }
  }
  next();
});
```

### ”页面”中获取登录状态

在“页面”中获取userId也很简单，使用计算属性是最好的，返回的userId具有响应性，这做的好处也是为了实时将登录状态反应到页面上，才不会出现显示已登录，但用户刷新一下才能知道登录状态已过期的尴尬情况。    

> 【PS】一些需要用户登录状态的api函数，也是通过这样的方法获取并使用。当然，建议在非首屏加载使用的api函数（比如提交表单），需要在调用前，检查一下userId还存不存在，以免出错。

```javascript
computed: {
  // 获取登录状态
  userId() {
    return this.$store.getters.getUserId;
  },
},
```

## 使用公共状态管理维护连接

有时会有需要取消请求的需求，比如上传文件耗时过长，用户不想等，这时候就必须有取消的功能。  
上篇提到的axios已经提供了一个基于CancelToken的取消机制，这里我们要配合vuex实现对全局连接的实时监控。  

> 【PS】示例的接口名称都是在写api函数的时候定义好的，想要更加灵活，可以每次请求都单独指定名字。

### 公共状态管理配置

示例中给公共状态下的com模块添加了cancelToken数组，用来保存发起的连接的名称和source（取消标记），并提供了如下三个mutations方法

```javascript
// 增加cancelToken
addCancelToken(_state, cancelToken) {
  _state.cancelToken.push(cancelToken);
  const arr = _state.cancelToken;
  _state.cancelToken = arr;
},
// 删除指定名字的cancelToken
deleteCancelToken(_state, name) {
  _state.cancelToken.some((i, index) => {
    if (i.name === name) {
      _state.cancelToken.splice(index, 1);
      return true;
    }
    return false;
  });
},
// 清空cancelToken
clearCancelToken(_state) {
  _state.cancelToken.forEach((i) => {
    if (i.source && typeof i.source.cancel === 'function') {
      i.source.cancel(`cancel${name}`);
    }
  });
  _state.cancelToken = [];
},
```

### 请求拦截器配置

基本思路：    

1.  在“请求发起前拦截器”中获取到source（取消标记），写入请求配置，并提交名称和source到公共状态管理；
2.  这时候通过查询公共状态中是否有这个名字的连接就可以获取到source进行取消；
3.  在“响应拦截器”中我们将已经成功的请求在公共状态管理中移除。

```javascript
// 请求发起前拦截器
myAxios.interceptors.request.use((_config) => {
  const config = _config;
  const source = axios.CancelToken.source();
  // 获取cancelToken
  config.cancelToken = source.token;

  // 取消请求标记保存
  store.commit('addCancelToken', {
    name: config.name,
    source,
  });

  return config;
}, () => {
  // 异常处理
  console.error('请求发起前拦截器异常');
});

// 响应拦截器
myAxios.interceptors.response.use((response) => {
  // 删除取消标记
  store.commit('deleteCancelToken', response.config.name);

  console.log(response);
  return response;
}, (error) => {
  console.error(error);
  // 清理取消标记
  store.commit('clearCancelToken');
  return Promise.reject(error);
});
```

### 代码中使用

使用计算属性获取cancelToken数组，这里是响应式的，所有我们其实可以知道现在有多少个请求还未完成了。  

> 【PS】实时对全局请求的监控已经实现，其实可以做的扩展就很多了，比如可以全局化loading的显示，只要数组内一直不为空持续一段时间就可以判断需要显示loading，如果为空则关闭loading。

```javascript
computed: {
  // 获取连接列表
  cancelToken() {
    return this.$store.state.com.cancelToken;
  },
},
```

遍历这个数组，查找到对应名字的连接就可以取消了。  

```javascript
this.cancelToken.forEach((i) => {
  console.log(i);
  if (i.name === '上传文件' && typeof i.source.cancel === 'function') {
    console.log('取消上传');
    i.source.cancel(`cancel${name}`);
  }
});
```

## 其他细节技巧

### 挂载常用工具到vue原型上减少引用

在初始化Vue的根组件前，给Vue的原型链上添加常用的工具，可以方便在vue文件中使用。这样做会影响所有Vue示例推荐只在单页面应用中使用。  
比如下面以我们的api集为例，这样在vue文件中`this.$api`就可以使用我们的api集，不需要重复引用。  

```javascript
// 将api模块挂载进vue方便在this调用
Vue.prototype.$api = api;
```

### 批量导入过滤器

在util文件夹下可以新建一个专门用来存过滤器的filter.js，然后批量导入的全局过滤器中。  

```javascript
import * as filters from './util/filter';
Object.keys(filters).forEach(k => Vue.filter(k, filters[k]));
```
