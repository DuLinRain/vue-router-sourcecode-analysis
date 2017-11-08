# 第四章 History类
Vue-Router默认工作在hash模式，但其实我们是可以认为指定它的工作模式的。在实例化VueRouter时需要传递options参数，options可以设置一个mode属性，用来配置VueRoute的路由模式。VueRoute支持三种路由模式，我们来看看官网的说明：

![](/assets/vue-router1.png)

![](https://i.imgur.com/cFXmXdg.png)

那么这几种模式是如何实现的呢？

实际上，这三种模式分别是通过三个类来实现的：

1. hash模式由HashHistory类实现。
2. history模式由HTML5History类实现。
3. abstract模式由AbstractHistory类实现。

而这三个类其实又都继承于History类。要想弄清楚这三种模式，首先得从他们的父类入手。

### 4.1 History类
History类定义在vue-router源码的1845 ~ 2143行，我们来具体看一下它的内容。

#### 4.1.1 成员属性
History类接收router和base两个参数，共有9个成员属性，分别是：

1. router。 表示VueRouter实例。实例化History类时的第一个参数。
2. base。 表示基路径。会用normalizeBase进行规范化。实例化History类时的第二个参数。
3. current。表示当前路由(route)。
4. pending。描述阻塞状态。
5. ready。描述是否就绪状态。
6. readyCbs。就绪状态的回调数组。
7. readyErrorCbs。就绪时产生错误的回调数组。
8. errorCbs。错误的回调数组
9. cb。监听时的回调函数。

其源码定义在1845 ~ 1855行：

	var History = function History (router, base) {
	  this.router = router;
	  this.base = normalizeBase(base);//规范化base
	  // start with a route object that stands for "nowhere"
	  this.current = START;//当前路径，初始化为/
	  this.pending = null;
	  this.ready = false;
	  this.readyCbs = [];
	  this.readyErrorCbs = [];
	  this.errorCbs = [];
	};

这里面咋一看只有八个成员属性，少了一个cb。实际上这个cb是在调用listen原型方法的时候定义的，我们稍后会讲到原型方法。

这里面用到了一个normalizeBase方法，该方法定义在vue-router源码的2009 ~ 2027行：

	//归一化base
	function normalizeBase (base) {
	  if (!base) {//如果没有提供base则从dom里面找base标签
	    if (inBrowser) {//如果在浏览器环境
	      // respect <base> tag
	      var baseEl = document.querySelector('base');
	      base = (baseEl && baseEl.getAttribute('href')) || '/';
	      // strip full URL origin
	      // 提取完整的URL
	      base = base.replace(/^https?:\/\/[^\/]+/, '');
	    } else {//非浏览器环境直接置为'/'
	      base = '/';
	    }
	  }
	  // make sure there's the starting slash
	  // 确保以斜线开头
	  if (base.charAt(0) !== '/') {
	    base = '/' + base;
	  }
	  // remove trailing slash
	  // 确保最后不以斜线结束
	  return base.replace(/\/$/, '')
	}

可以看到，该函数主要是用于对提供的base进行规范化处理。这里面又分几种情况：

- 如果没有提供base参数，则判断是否在浏览器环境中，如果在，则从DOM中查找base标签，如果找到了base标签，则拿他的href，否则用'/'作为base。然后根据此时的base去除主机和域名，拿到真正的base。比如'http://www.baidu.com/a/b/c'.replace(/^https?:\/\/[^\/]+/, '');拿到'/a/b/c'。如果不在浏览器环境，则将base设置为'/'
- 如果提供了base，则判断是不是以'/'开头，如果不是，则加上'/'
- 然后判断base是不是以'/'结尾，如果是，则去掉结尾的'/'

也就是说，如果base是'/'，会返回一个''（空字符串）

#### 4.1.2 原型方法
History类定义了`listen`、`onReady`、`onError`、`transitionTo`、`confirmTransition`、`updateRoute` 6个原型方法。我们分别来看一下它的实现。

##### 4.1.2.1 History.prototype.listen方法
History.prototype.listen方法定义在vue-router源码的1857 ~ 1859行，通过它来监听路由的变化，并且会更新实例上的cb属性。其源码如下：

	History.prototype.listen = function listen (cb) {
	  this.cb = cb;//callback属性
	};

##### 4.1.2.2 History.prototype.onReady方法
History.prototype.onReady方法定义在vue-router源码的1861 ~ 1870行，它接收cb, errorCb两个参数，用于注册ready时的回调。当History实例的当前ready状态为true时，会执行cb回调，否则将cb回调函数放入实例的readyCbs队列。如果提供了errorCb参数，则将errorCb加入readyErrorCbs队列。其源码如下：

	History.prototype.onReady = function onReady (cb, errorCb) {
	  if (this.ready) {
	    cb();
	  } else {
	    this.readyCbs.push(cb);
	    if (errorCb) {
	      this.readyErrorCbs.push(errorCb);
	    }
	  }
	};

##### 4.1.2.3 History.prototype.onError方法
History.prototype.onError方法定义在vue-router源码的1872 ~ 1874行，它接收一个errorCb参数，用于注册错误的回调。它会把errorCb加入到History实例的errorCbs队列。其源码如下：

	History.prototype.onError = function onError (errorCb) {
	  this.errorCbs.push(errorCb);
	};

##### 4.1.2.4 History.prototype.transitionTo方法
History.prototype.transitionTo方法定义在vue-router源码的1876 ~ 1899行，它主要用于路由跳转时调用。接收location, onComplete, onAbort三个参数，分别表示目标位置、完成时回调、中止时回调。transitionTo方法实际上是先拿当前路由和location进行路由匹配，当匹配到对应的目标route后，调用confirmTransition完成真正的跳转操作，confirmTransition方法的源码我们稍后会讲。transitionTo方法的源码如下：

	History.prototype.transitionTo = function transitionTo (location, onComplete, onAbort) {
	    var this$1 = this;
	
	  var route = this.router.match(location, this.current);
	  this.confirmTransition(route, function () {//成功时的callback
	    this$1.updateRoute(route);
	    onComplete && onComplete(route);
	    this$1.ensureURL();
	
	    // fire ready cbs once
	    if (!this$1.ready) {
	      this$1.ready = true;
	      this$1.readyCbs.forEach(function (cb) { cb(route); });
	    }
	  }, function (err) {//失败时的callback
	    if (onAbort) {
	      onAbort(err);
	    }
	    if (err && !this$1.ready) {
	      this$1.ready = true;
	      this$1.readyErrorCbs.forEach(function (cb) { cb(err); });
	    }
	  });
	};

##### 4.1.2.5 History.prototype.confirmTransition方法
History.prototype.confirmTransition方法定义在vue-router源码的1901 ~ 1998行，它是真正完成路由跳转的函数。它接收route, onComplete, onAbort三个参数，分别表示目标路由、完成时回调、中止时回调。我们来看看它的源码：

	History.prototype.confirmTransition = function confirmTransition (route, onComplete, onAbort) {
	    var this$1 = this;
	
	  var current = this.current;
	  var abort = function (err) {
	    if (isError(err)) {
	      if (this$1.errorCbs.length) {
	        this$1.errorCbs.forEach(function (cb) { cb(err); });
	      } else {
	        warn(false, 'uncaught error during route navigation:');
	        console.error(err);
	      }
	    }
	    onAbort && onAbort(err);
	  };
	  if (
	    isSameRoute(route, current) &&
	    // in the case the route map has been dynamically appended to
	    route.matched.length === current.matched.length
	  ) {
	    this.ensureURL();
	    return abort()
	  }
	
	  var ref = resolveQueue(this.current.matched, route.matched);
	    var updated = ref.updated;
	    var deactivated = ref.deactivated;
	    var activated = ref.activated;
	
	  var queue = [].concat(
	    // in-component leave guards
	    extractLeaveGuards(deactivated),
	    // global before hooks
	    this.router.beforeHooks,
	    // in-component update hooks
	    extractUpdateHooks(updated),
	    // in-config enter guards
	    activated.map(function (m) { return m.beforeEnter; }),
	    // async components
	    resolveAsyncComponents(activated)
	  );
	
	  this.pending = route;// 导航被触发(对应第1步)
	  var iterator = function (hook, next) {
	    if (this$1.pending !== route) {
	      return abort()
	    }
	    try {
	      hook(route, current, function (to) {
	        if (to === false || isError(to)) {
	          // next(false) -> abort navigation, ensure current URL
	          this$1.ensureURL(true);
	          abort(to);
	        } else if ( 
	          typeof to === 'string' ||
	          (typeof to === 'object' && (
	            typeof to.path === 'string' ||
	            typeof to.name === 'string'
	          ))
	        ) {
	          // next('/') or next({ path: '/' }) -> redirect
	          abort();
	          if (typeof to === 'object' && to.replace) {
	            this$1.replace(to);
	          } else {
	            this$1.push(to);
	          }
	        } else {
	          // confirm transition and pass on the value
	          next(to);
	        }
	      });
	    } catch (e) {
	      abort(e);
	    }
	  };
	
	  runQueue(queue, iterator, function () {
	    var postEnterCbs = [];
	    var isValid = function () { return this$1.current === route; };
	    // wait until async components are resolved before
	    // extracting in-component enter guards
	    var enterGuards = extractEnterGuards(activated, postEnterCbs, isValid);
	    var queue = enterGuards.concat(this$1.router.resolveHooks);
	    runQueue(queue, iterator, function () {
	      if (this$1.pending !== route) {
	        return abort()
	      }
	      this$1.pending = null;
	      onComplete(route);
	      if (this$1.router.app) {
	        this$1.router.app.$nextTick(function () {
	          postEnterCbs.forEach(function (cb) { cb(); });
	        });
	      }
	    });
	  });
	};

confirmTransition是一个非常重要的函数，内容也比较多，我们来一段段地看。

confirmTransition会先判断当前路由和目标路由是否同一个路由，如果是则中止跳转。会调用传递的abort方法。abort定义在confirmTransition函数内部：

      var abort = function (err) {
	    if (isError(err)) {
	      if (this$1.errorCbs.length) {
	        this$1.errorCbs.forEach(function (cb) { cb(err); });
	      } else {
	        warn(false, 'uncaught error during route navigation:');
	        console.error(err);
	      }
	    }
	    onAbort && onAbort(err);
	  };

我们看到，他主要做的是执行history实例上errorCbs数组中注册的错误回调。当errorCbs数组中注册的错误回调执行完成之后再执行用户调用transitionTo方法传递的错误回调函数。

随后，confirmTransition函数会拿当前路由和目标路由进行解析，解析出**失活的组件**、**重用的组件**、**激活的组件**：

	var ref = resolveQueue(this.current.matched, route.matched);
    var updated = ref.updated;// 重用的组件
    var deactivated = ref.deactivated;// 失活的组件
    var activated = ref.activated;// 激活的组件

后面的代码就涉及到一个非常重要的概念了——**导航解析流程**。[官方文档](https://router.vuejs.org/zh-cn/advanced/navigation-guards.html)对此有详尽的描述：

![](/assets/2-12.png)

![](https://i.imgur.com/BL2ZLyb.png)

继续看confirmTransition函数，我们可以看到，它将前6个步鄹的导航钩子提取到一个队列里，然后执行该队列：

	var queue = [].concat(
	    // in-component leave guards
	    extractLeaveGuards(deactivated),//对应第2步——提取失活的组件的离开守卫
	    // global before hooks
	    this.router.beforeHooks, // 对应第3步——全局的 beforeEach 守卫
	    // in-component update hooks
	    extractUpdateHooks(updated),// 对应第4步——重用的组件里 beforeRouteUpdate 守卫
	    // in-config enter guards
	    activated.map(function (m) { return m.beforeEnter; }), // 对应第5步——路由配置里 beforeEnter 守卫
	    // async components
	    resolveAsyncComponents(activated) // 对应第6步——提取异步路由组件守卫
	);

队列是调用runQueue函数执行的，runQueue函数定义在源码的1717 ~ 1732行，它所做的相当于是遍历数组queue，将数组元素作为fn的参数执行响应的函数，当数组遍历完后执行响应的回调cb:

	function runQueue (queue, fn, cb) {
	  var step = function (index) {
	    if (index >= queue.length) {
	      cb();
	    } else {
	      if (queue[index]) {
	        fn(queue[index], function () {
	          step(index + 1);
	        });
	      } else {
	        step(index + 1);
	      }
	    }
	  };
	  step(0);
	}

runQueue的实际调用如下：

	runQueue(queue, iterator, function () {
	    var postEnterCbs = [];
	    var isValid = function () { return this$1.current === route; };
	    // wait until async components are resolved before
	    // extracting in-component enter guards
	    var enterGuards = extractEnterGuards(activated, postEnterCbs, isValid);
	    var queue = enterGuards.concat(this$1.router.resolveHooks);
	    runQueue(queue, iterator, function () {
	      if (this$1.pending !== route) {
	        return abort()
	      }
	      this$1.pending = null;
	      onComplete(route);
	      if (this$1.router.app) {
	        this$1.router.app.$nextTick(function () {
	          postEnterCbs.forEach(function (cb) { cb(); });
	        });
	      }
	    });
	});

实际上它就是遍历前面的定义的数组queue（包含导航流程的前6步），将这每种导航守卫作为参数调用iterator，iterator的代码也定义在confirmTransition函数内：

	var iterator = function (hook, next) {
	    if (this$1.pending !== route) {
	      return abort()
	    }
	    try {
	      hook(route, current, function (to) {
	        if (to === false || isError(to)) {
	          // next(false) -> abort navigation, ensure current URL
	          this$1.ensureURL(true);
	          abort(to);
	        } else if ( 
	          typeof to === 'string' ||
	          (typeof to === 'object' && (
	            typeof to.path === 'string' ||
	            typeof to.name === 'string'
	          ))
	        ) {
	          // next('/') or next({ path: '/' }) -> redirect
	          abort();
	          if (typeof to === 'object' && to.replace) {
	            this$1.replace(to);
	          } else {
	            this$1.push(to);
	          }
	        } else {
	          // confirm transition and pass on the value
	          next(to);
	        }
	      });
	    } catch (e) {
	      abort(e);
	    }
	};

它所完成的工作就是[官方文档](https://router.vuejs.org/zh-cn/advanced/navigation-guards.html)所描述的：

![](/assets/2-13.png)

![](https://i.imgur.com/bi13m3J.png)

在runQueue执行完，回调调用前，实际上也就完成了**完整的导航解析流程**的前6步：

![](/assets/2-14.png)

![](https://i.imgur.com/IjKFc96.png)

剩余的导航流程是在runQueue的回调函数中完成的，代码前面已经粘贴过，这里我们再次单独拎出来看看：

	var postEnterCbs = [];
    var isValid = function () { return this$1.current === route; };
    // wait until async components are resolved before
    // extracting in-component enter guards
    var enterGuards = extractEnterGuards(activated, postEnterCbs, isValid);// 被激活的组件里 beforeRouteEnter 守卫
    var queue = enterGuards.concat(this$1.router.resolveHooks);// 全局的 beforeResolve 守卫
    runQueue(queue, iterator, function () {
      if (this$1.pending !== route) {
        return abort()
      }
      this$1.pending = null;
      onComplete(route);// 导航被确认。
      if (this$1.router.app) {
        this$1.router.app.$nextTick(function () {
          postEnterCbs.forEach(function (cb) { cb(); });
        });
      }
    });

我们看到它实际上提取了两种守卫——**被激活的组件里 beforeRouteEnter 守卫(对应第7步)**和**全局的 beforeResolve 守卫(对应第8步)**。

然后第二次调用runQueue执行队列，执行完后在回调函数中执行onComplete方法以**确认导航(对应第9步)**。

在执行onComplete方法后一定会执行updateRoute方法，我们看看transitionTo中调用confirmTransition时的回调(1880 ~ 1898行)予以确认一下：

	this.confirmTransition(route, function () {
	    this$1.updateRoute(route);//更新路由
	    onComplete && onComplete(route);
	    this$1.ensureURL();
	
	    // fire ready cbs once
	    if (!this$1.ready) {
	      this$1.ready = true;
	      this$1.readyCbs.forEach(function (cb) { cb(route); });
	    }
	  }, function (err) {
	    if (onAbort) {
	      onAbort(err);
	    }
	    if (err && !this$1.ready) {
	      this$1.ready = true;
	      this$1.readyErrorCbs.forEach(function (cb) { cb(err); });
	    }
	});

而执行updateRoute函数就必定会执行全局的 afterEach 钩子（**对应第10步**），可以看看History.prototype.updateRoute方法的实现就知道了：

	History.prototype.updateRoute = function updateRoute (route) {
	  var prev = this.current;
	  this.current = route;
	  this.cb && this.cb(route);
	  this.router.afterHooks.forEach(function (hook) {
	    hook && hook(route, prev);
	  });
	};

最后会调用nextTick(**对应第11步**)，执行beforeRouteEnter 守卫中传给 next 的回调函数（**对应第12步**）：

	if (this$1.router.app) {
        this$1.router.app.$nextTick(function () {//
          postEnterCbs.forEach(function (cb) { cb(); });
        });
    }

其中的postEnterCbs是在调用extractEnterGuards方法提取beforeRouteEnter 守卫时赋值的。

至此，第二个runQueue的调用完成了剩余的全部导航流程：

![](/assets/2-15.png)

![](https://i.imgur.com/GUkZnfo.png)

##### 4.1.2.6 History.prototype.updateRoute方法
History.prototype.updateRoute方法定义在vue-router源码的2000 ~ 2007行，用于更新路由。 显然，更新路由一般发生在路由跳转完成之后，也就是confirmTransition的成功回调(onComplete)里面, 它所做的主要就是将目标路由赋给current，完成更新。当然还需要调用history实例上的回调cb，除此之外还得调用router实例上注册的后置钩子(afterHooks)函数。

	History.prototype.updateRoute = function updateRoute (route) {
	  var prev = this.current;
	  this.current = route;
	  this.cb && this.cb(route);
	  this.router.afterHooks.forEach(function (hook) {
	    hook && hook(route, prev);
	  });
	};

