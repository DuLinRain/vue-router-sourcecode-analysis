# 第五章 HTML5History类
### 5.1 HTML5History类源码实现
HTML5History类定义在vue-router源码的2143 ~ 2226行，采用的是经典的IIFE(立即调用函数表达式)方式实现的：

	//HTML5History继承自History
	//HTML5History模式下支持滚动行为
	var HTML5History = (function (History$$1) {
	  function HTML5History (router, base) {
	    var this$1 = this;
	
	    History$$1.call(this, router, base);//调用父类的构造函数
	
	    var expectScroll = router.options.scrollBehavior;
	
	    if (expectScroll) {
	      setupScroll();
	    }
	
	    var initLocation = getLocation(this.base);
	    window.addEventListener('popstate', function (e) {
	      var current = this$1.current;
	
	      // Avoiding first `popstate` event dispatched in some browsers but first
	      // history route not updated since async guard at the same time.
	      var location = getLocation(this$1.base);
	      if (this$1.current === START && location === initLocation) {
	        return
	      }
	
	      this$1.transitionTo(location, function (route) {
	        if (expectScroll) {
	          handleScroll(router, route, current, true);
	        }
	      });
	    });
	  }
	  //下面三步是经典的类继承的实现
	  if ( History$$1 ) HTML5History.__proto__ = History$$1;
	  //实现HTML5History.prototype.__proto__ = History$$1.prototype
	  HTML5History.prototype = Object.create( History$$1 && History$$1.prototype );
	  //修正构造函数
	  HTML5History.prototype.constructor = HTML5History;
	  //定义原型函数go
	  HTML5History.prototype.go = function go (n) {
	    window.history.go(n);
	  };
	  //定义原型函数push
	  HTML5History.prototype.push = function push (location, onComplete, onAbort) {
	    var this$1 = this;
	
	    var ref = this;
	    var fromRoute = ref.current;
	    this.transitionTo(location, function (route) {
	      pushState(cleanPath(this$1.base + route.fullPath));
	      handleScroll(this$1.router, route, fromRoute, false);
	      onComplete && onComplete(route);
	    }, onAbort);
	  };
	  //定义原型函数replace
	  HTML5History.prototype.replace = function replace (location, onComplete, onAbort) {
	    var this$1 = this;
	
	    var ref = this;
	    var fromRoute = ref.current;
	    this.transitionTo(location, function (route) {
	      replaceState(cleanPath(this$1.base + route.fullPath));
	      handleScroll(this$1.router, route, fromRoute, false);
	      onComplete && onComplete(route);
	    }, onAbort);
	  };
	  //定义原型函数ensureURL
	  HTML5History.prototype.ensureURL = function ensureURL (push) {
	    if (getLocation(this.base) !== this.current.fullPath) {
	      var current = cleanPath(this.base + this.current.fullPath);
	      push ? pushState(current) : replaceState(current);
	    }
	  };
	  //定义原型函数getCurrentLocation
	  HTML5History.prototype.getCurrentLocation = function getCurrentLocation () {
	    return getLocation(this.base)
	  };
	
	  return HTML5History;//返回这个类
	}(History));

该类继承自History类，继承的方式也是经典的类继承方式，我们来大体看一下：

该函数的构造函数中首先借用父类的构造函数（2147行）：

	History$$1.call(this, router, base);//调用父类的构造函数

然后实现原型继承（2174 ~ 2176行）：

	//下面三步是经典的类继承的实现
	  if ( History$$1 ) HTML5History.__proto__ = History$$1;
	  //实现HTML5History.prototype.__proto__ = History$$1.prototype
	  HTML5History.prototype = Object.create( History$$1 && History$$1.prototype );
	  //修正构造函数
	  HTML5History.prototype.constructor = HTML5History;

这样就完成了从History类继承。

我们知道，HTML5History模式支持滚动行为，所以在构造函数中有这么一段代码（2149 ~ 2171行）处理用户的滚动行为：

	var expectScroll = router.options.scrollBehavior;
	
    if (expectScroll) {
      setupScroll();
    }

    var initLocation = getLocation(this.base);
    window.addEventListener('popstate', function (e) {
      var current = this$1.current;

      // Avoiding first `popstate` event dispatched in some browsers but first
      // history route not updated since async guard at the same time.
      var location = getLocation(this$1.base);
      if (this$1.current === START && location === initLocation) {
        return
      }

      this$1.transitionTo(location, function (route) {
        if (expectScroll) {
          handleScroll(router, route, current, true);
        }
      });
    });

关于HTML5History模式的详细内容在后面的小节介绍。

HTML5History类定义了一些与浏览器操作有关的原型函数，我们分别来看一下。

#### 5.1.1 原型函数

##### 5.1.1.1 HTML5History.prototype.go
HTML5History.prototype.go方法定义在2178 ~ 2180行，它实际上调用的就是原生的window.history.go(n)来实现的：

	//定义原型函数go
	HTML5History.prototype.go = function go (n) {
	    window.history.go(n);
	};

##### 5.1.1.2 HTML5History.prototype.push
HTML5History.prototype.push方法定义在源码的2182 ~ 2192行，该方法用于HTML5History模式下将当新路由加入到history中。因为是 HTML5History模式，需要支持滚动，所以它在调用transitionTo方法的成功回调中会调用handleScroll方法：

	  //定义原型函数push
	  HTML5History.prototype.push = function push (location, onComplete, onAbort) {
	    var this$1 = this;
	
	    var ref = this;
	    var fromRoute = ref.current;
	    this.transitionTo(location, function (route) {
	      pushState(cleanPath(this$1.base + route.fullPath));
	      handleScroll(this$1.router, route, fromRoute, false);
	      onComplete && onComplete(route);
	    }, onAbort);
	  };

##### 5.1.1.3 HTML5History.prototype.replace
HTML5History.prototype.replace定义在源码的2194 ~ 2204行，该方法用于HTML5History模式下将新路由替换history中当前路由。因为是 HTML5History模式，需要支持滚动，所以它在调用transitionTo方法的成功回调中会调用handleScroll方法：

	//定义原型函数replace
	  HTML5History.prototype.replace = function replace (location, onComplete, onAbort) {
	    var this$1 = this;
	
	    var ref = this;
	    var fromRoute = ref.current;
	    this.transitionTo(location, function (route) {
	      replaceState(cleanPath(this$1.base + route.fullPath));
	      handleScroll(this$1.router, route, fromRoute, false);
	      onComplete && onComplete(route);
	    }, onAbort);
	  };

##### 5.1.1.4 HTML5History.prototype.ensureURL
HTML5History.prototype.ensureURL定义在vue-router源码的2206 ~ 2211行：

	HTML5History.prototype.ensureURL = function ensureURL (push) {
	    if (getLocation(this.base) !== this.current.fullPath) {
	      var current = cleanPath(this.base + this.current.fullPath);
	      push ? pushState(current) : replaceState(current);
	    }
	};

通过getLocation从window.location.pathname中拿去除base后的完整路径，与current属性上的fullPath作比较，如果二者不一致说明URL发生了变化，然后调用cleanPath进行进化(主要就是去除两个连续的/)后，跟新state,具体是增加还是替换取决于传递的参数:

	HTML5History.prototype.ensureURL = function ensureURL (push) {
	    if (getLocation(this.base) !== this.current.fullPath) {
	      var current = cleanPath(this.base + this.current.fullPath);
	      push ? pushState(current) : replaceState(current);
	    }
	};

getLocation定义在2220 ~ 2226行：

	function getLocation (base) {
	  var path = window.location.pathname;
	  if (base && path.indexOf(base) === 0) {
	    path = path.slice(base.length);
	  }
	  return (path || '/') + window.location.search + window.location.hash
	}

cleanPath定义在645 ~ 647行：

	//净化path，主要就是将连续两个//换成一个
	//'a//b///c/d//f'.replace(/\/\//g, '/') "a/b//c/d/f"
	//并没法完全净化，只会净化一次
	function cleanPath (path) {
	    return path.replace(/\/\//g, '/')
	}

##### 5.1.1.5 HTML5History.prototype.getCurrentLocation
HTML5History.prototype.getCurrentLocation定义在2213 ~ 2215行，用于拿当前URL的path, 包含去掉base后的path, search, 和hash部分
	HTML5History.prototype.getCurrentLocation = function getCurrentLocation () {
	    return getLocation(this.base)
	};

实际调用的还是getLocation，getLocation我们在前面已经介绍过了。

### 5.2 HTML5History模式是如何支持滚动行为的？
我们知道，vue-router支持三种history模式，其中在使用HTML5 history 模式时可以支持**滚动行为**(注： 从源码来看，hash模式应该也是支持滚动行为的)，那么什么是滚动行为呢？ 官方是这么说的：

>使用前端路由，当切换到新路由时，想要页面滚到顶部，或者是保持原先的滚动位置，就像重新加载页面那样。 vue-router 能做到，而且更好，它让你可以自定义路由切换时页面如何滚动。

关于滚动行为的介绍与用法，官方文档有详细的描述，这里我们直接粘贴过来看看：

#### 5.2.1 vue-router官方文档关于滚动行为的介绍
##
当创建一个 Router 实例，你可以提供一个 scrollBehavior 方法：

	const router = new VueRouter({
	  routes: [...],
	  scrollBehavior (to, from, savedPosition) {
	    // return 期望滚动到哪个的位置
	  }
	})
	
scrollBehavior 方法接收 to 和 from 路由对象。第三个参数 savedPosition 当且仅当 popstate 导航 (通过浏览器的 前进/后退 按钮触发) 时才可用。

这个方法返回滚动位置的对象信息，长这样：

- { x: number, y: number }
- { selector: string, offset? : { x: number, y: number }} (offset 只在 2.6.0+ 支持)

如果返回一个 falsy (译者注：falsy 不是 false，[参考这里](https://developer.mozilla.org/zh-CN/docs/Glossary/Falsy))的值，或者是一个空对象，那么不会发生滚动。

举例：

	scrollBehavior (to, from, savedPosition) {
	  return { x: 0, y: 0 }
	}

对于所有路由导航，简单地让页面滚动到顶部。

返回 savedPosition，在按下 后退/前进 按钮时，就会像浏览器的原生表现那样：

	scrollBehavior (to, from, savedPosition) {
	  if (savedPosition) {
	    return savedPosition
	  } else {
	    return { x: 0, y: 0 }
	  }
	}

如果你要模拟『滚动到锚点』的行为：

	scrollBehavior (to, from, savedPosition) {
	  if (to.hash) {
	    return {
	      selector: to.hash
	    }
	  }
	}

我们还可以利用[路由元信息](https://router.vuejs.org/zh-cn/advanced/meta.html)更细颗粒度地控制滚动。查看完整例子请[移步这里](https://github.com/vuejs/vue-router/blob/next/examples/scroll-behavior/app.js)。

##

整个文档的意思就是说，在HTML5 history 模式时，通过在路由实例化时提供一个scrollBehavior选项来指定在路由切换时页面的滚动行为。如果你希望发生滚动行为，该函数必须返回一个对象，这个对象必须包含以下两种格式的信息：

- { x: number, y: number }
- { selector: string, offset? : { x: number, y: number }} (offset 只在 2.6.0+ 支持)

而如果你不希望发生滚动行为，那么你只需要给它返回一个falsy类型的值就可以了。

而返回savedPosition则在按下 后退/前进 按钮时，就会像浏览器的原生表现那样：

	scrollBehavior (to, from, savedPosition) {
	  if (savedPosition) {
	    return savedPosition
	  } else {
	    return { x: 0, y: 0 }
	  }
	}

这句话什么意思呢？就是说，你正常的点击页面的方式进行路由导航时，savedPosition值是null, 是不会返回savedPosition的，而当你点击后退/前进 按钮时，此时savedPosition值不为null，一定会返回savedPosition。

我们举个例子来看看：

![](/assets/1.png)

![](https://i.imgur.com/GvCSXn4.png)

这个例子定义了四个路由，当直接点击链接切换路由的时候，savedPosition总是null，而当点击浏览器的前进/后退按钮时，savedPosition不为空，页面会滚动到对应的位置。

我们现在已经知道了路由切换回进行滚动行为，那么我们在来看看路由是如何切换的呢？也就是说当用户点击链接后，路由是如何实现的呢? 实际上，HTML5 History模式下主要采用的是两个方法和一个事件：

- history.pushState()
- history.replaceState()
- window.onpopstate事件

前两个方法都是在HTML5中才引入的，我们来具体了解一下他们，这里直接拿MDN文档贴过来：

##

#### 5.2.2 HTML5 规范中关于history的介绍

##### 5.2.2.1 pushState() 方法

在 HTML 文件中,  history.pushState() 方法向浏览器历史添加了一个状态。

pushState() 带有三个参数：一个状态对象，一个标题（现在被忽略了），以及一个可选的URL地址。下面将对这三个参数进行细致的检查：

- state object — 状态对象是一个由 pushState()方法创建的、与历史纪录相关的JS对象。当用户定向到一个新的状态时，会触发popstate事件。事件的state属性包含了历史纪录的state对象。（译者注：总而言之，它存储JSON字符串，可以用在popstate事件中。）state 对象可以是任何可以序列化的东西。由于 火狐 会将这些对象存储在用户的磁盘上，所以用户在重启浏览器之后这些state对象会恢复，我们施加一个最大640k 的字符串在state对象的序列化表示上。如果你像pushState() 方法传递了一个序列化表示大于640k 的state对象，这个方法将扔出一个异常。如果你需要更多的空间，推荐使用sessionStorage或者localStorage。
- title — 火狐浏览器现在已经忽略此参数，将来也许可能被使用。考虑到将来有可能的改变，传递一个空字符串是安全的做法。当然，你可以传递一个短标题给你要转变成的状态。（译者注：现在大多数浏览器不支持或者忽略这个参数，最好用null代替）
- URL — 这个参数提供了新历史纪录的地址。请注意，浏览器在调用pushState()方法后不会去加载这个URL，但有可能在之后会这样做，比如用户重启浏览器之后。新的URL不一定要是绝对地址，如果它是相对的，它一定是相对于当前的URL。新URL必须和当前URL在同一个源下;否则，pushState() 将丢出异常。这个参数可选，如果它没有被特别标注，会被设置为文档的当前URL。

一些情况下，调用pushState和设置 window.location = "#foo"相当，这种状况下，这两种行为都会创建和激活另一个和当前页面有关的历史纪录。但是pushState()有其他优势：

- 新URL可以是当前URL同源下的任意地址。相反的，设置window.location会让你保持在相同页面，除非你只修改hash.
- 如果不必要，你可以不改变URL，相反的，将window.location设定为“#foo”；只会创建一个新的历史纪录，如果当前hash不为#foo.
- 你可以关联任意的数据到你的新历史纪录中。使用基于hash的方法，你需要将所有相关 的数据编码为一个短字符串。

**请注意**
>pushState()方法绝不会导致hashchange 事件被激活，即使新的URL和旧的只在hash上有区别。

**语法：**

	history.pushState(state, title, url);

**示例：**

创建了一个新的由 state, title, 和 url设定的浏览器历史纪录.

	var state = { 'page_id': 1, 'user_id': 5 };
	var title = 'Hello World';
	var url = 'hello-world.html';
	
	history.pushState(state, title, url);

##### 5.2.2.2 replaceState() 方法
history.replaceState() 的使用与 history.pushState() 非常相似，区别在于  replaceState()  是修改了当前的历史记录项而不是新建一个。 注意这并不会阻止其在全局浏览器历史记录中创建一个新的历史记录项。

replaceState() 的使用场景在于为了响应用户操作，你想要更新状态对象state或者当前历史记录的URL。

**replaceState() 方法示例**

假设 http://mozilla.org/foo.html 执行了如下JavaScript代码：

	var stateObj = { foo: "bar" };
	history.pushState(stateObj, "page 2", "bar.html");
        
上文2行代码可以在 "replaceState() 方法示例" 部分找到。然后，假设http://mozilla.org/bar.html执行了如下 JavaScript：

	history.replaceState(stateObj, "page 3", "bar2.html");
        
这将会导致地址栏显示http://mozilla.org/bar2.html,，但是浏览器并不会去加载bar2.html 甚至都不需要检查 bar2.html 是否存在。

假设现在用户重新导向到了http://www.microsoft.com，然后点击了回退按按钮。这里，地址栏会显示http://mozilla.org/bar2.html。加入用户再次点击回退按钮，地址栏会显示http://mozilla.org/foo.html，完全跳过了abar.html。

##### 5.2.2.2 popstate 事件

window.onpopstate是popstate事件在window对象上的事件处理程序.

每当处于激活状态的历史记录条目发生变化时,popstate事件就会在对应window对象上触发. 如果当前处于激活状态的历史记录条目是由history.pushState()方法创建,或者由history.replaceState()方法修改过的, 则popstate事件对象的state属性包含了这个历史记录条目的state对象的一个拷贝.

调用history.pushState()或者history.replaceState()不会触发popstate事件. popstate事件只会在浏览器某些行为下触发, 比如点击后退、前进按钮(或者在JavaScript中调用history.back()、history.forward()、history.go()方法).

当网页加载时,各浏览器对popstate事件是否触发有不同的表现,Chrome 和 Safari会触发popstate事件, 而Firefox不会.

**语法**

	window.onpopstate = funcRef;
	funcRef 是个函数名.

**popstate事件**

假如当前网页地址为http://example.com/example.html,则运行下述代码后:

	window.onpopstate = function(event) {
	  alert("location: " + document.location + ", state: " + JSON.stringify(event.state));
	};
	//绑定事件处理函数. 
	//添加并激活一个历史记录条目 http://example.com/example.html?page=1,条目索引为1
	history.pushState({page: 1}, "title 1", "?page=1");
	//添加并激活一个历史记录条目 http://example.com/example.html?page=2,条目索引为2    
	history.pushState({page: 2}, "title 2", "?page=2"); 
	//修改当前激活的历史记录条目 http://ex..?page=2 变为 http://ex..?page=3,条目索引为3   
	history.replaceState({page: 3}, "title 3", "?page=3"); 
	history.back(); // 弹出 "location: http://example.com/example.html?page=1, state: {"page":1}"
	history.back(); // 弹出 "location: http://example.com/example.html, state: null
	history.go(2);  // 弹出 "location: http://example.com/example.html?page=3, state: {"page":3}

即便进入了那些非pushState和replaceState方法作用过的(比如http://example.com/example.html)没有state对象关联的那些网页, popstate事件也仍然会被触发.

##

#### 5.3 HTML5 History模式在Vue-router中是如何使用的？
我们看到，不管是pushState还是replaceState，第一个参数都是一个state对象，相当于是作为第三个参数的key，存储着k-v对。我么来看看vue-router是如何实现的。

vue-router在源码的1680行定义了`_Key`：

	var _key = genKey();

可以看到它实际上调用的是genKey()，genKey()定义在vuex源码的1682 ~ 1684行：

	function genKey () {//拿当前(代码首次运行)时间作为key
	  return Time.now().toFixed(3)
	}

可以看到它实际上是以genKey执行时的当前时间作为`_key`，这里的时间会优先使用window.perfermence性能API，如源码1676 ~ 1678行所示：

	var Time = inBrowser && window.performance && window.performance.now
	  ? window.performance
	  : Date;

在源码的1686 ~ 1692行定义个方法，分别用于获取`_Key`和更新`_Key`:

	function getStateKey () {
	  return _key
	}
	//重新设置key
	function setStateKey (key) {
	  _key = key;
	}

vue-router源码还在1694 ~ 1709行定义了pushState方法对原生的方法进行了封装，接收两个参数，第二个用于指定是否是replaceState:

	//Safari需要特殊处理
	function pushState (url, replace) {
	  saveScrollPosition();
	  // try...catch the pushState call to get around Safari
	  // DOM Exception 18 where it limits to 100 pushState calls
	  var history = window.history;
	  try {
	    if (replace) {
	      history.replaceState({ key: _key }, '', url);
	    } else {
	      _key = genKey();//重新生成key
	      history.pushState({ key: _key }, '', url);
	    }
	  } catch (e) {
	    window.location[replace ? 'replace' : 'assign'](url);
	  }
	}

该函数和原生pushState的套路是一样的，特别之处是该函数针对Safari浏览器做了特殊的处理。因为Safari对pushState的数量有限制，所以当超过这个数量之后无法再调用pushState，改为调用window.location.replace或window.location.assign。

每当执行pushState的时候会首先调用saveScrollPosition函数，saveScrollPosition函数定义在1589 ~ 1597行，用于保存当前页面的滚动位置：

	function saveScrollPosition () {
	  var key = getStateKey();//保存对应的位置
	  if (key) {
	    positionStore[key] = {
	      x: window.pageXOffset,
	      y: window.pageYOffset
	    };
	  }
	}

位置信息是以k-v形式保存在positionStore对象中的，positionStore定义在源码的1534行：

	var positionStore = Object.create(null);

除此之外，源码还在1711 ~ 1713行定义了replaceState方法，实际上就是调用的pushState方法，只不过将第二个参数置为true：

	//替换状态
	function replaceState (url) {
	  pushState(url, true);
	}

说了这么多还是没有具体到HTML5History模式下路由是如何工作的，现在我们来看一下。

在源码2143 ~ 2172行定义了HTML5History类的构造函数：

	  function HTML5History (router, base) {
	    var this$1 = this;
	
	    History$$1.call(this, router, base);//调用父类的构造函数
	
	    var expectScroll = router.options.scrollBehavior;
	
	    if (expectScroll) {
	      setupScroll();
	    }
	
	    var initLocation = getLocation(this.base);
	    window.addEventListener('popstate', function (e) {
	      var current = this$1.current;
	
	      // Avoiding first `popstate` event dispatched in some browsers but first
	      // history route not updated since async guard at the same time.
	      var location = getLocation(this$1.base);
	      if (this$1.current === START && location === initLocation) {
	        return
	      }
	
	      this$1.transitionTo(location, function (route) {
	        if (expectScroll) {
	          handleScroll(router, route, current, true);
	        }
	      });
	    });
	  }

这个类的构造函数中会先判断用户在实例化router的时候有没有传递scrollBehavior方法，如果有传递的话，会先调用setupScroll方法，setupScroll方法定义在vue-router源码的1536 ~ 1545行：

	function setupScroll () {
	  // Fix for #1585 for Firefox
	  window.history.replaceState({ key: getStateKey() }, '');
	  window.addEventListener('popstate', function (e) {
	    saveScrollPosition();
	    if (e.state && e.state.key) {
	      setStateKey(e.state.key);
	    }
	  });
	}

该方法会第一次注册popstate事件，保存当前位置，然后如果事件存在key的话，更新`_Key`。 

随后构造函数继续，根据实例化router时传递的base拿到初始位置initLocation，然后注册第二个popstate事件，该事件响应函数会将初始位置和当前位置进行对比，如果不相同会调用transitionTo执行相应的路由跳转。 而在跳转的成功回调函数中会调用，如果实例化router时传递了scrollBehavior则会处理滚动：

	if (expectScroll) {
	  handleScroll(router, route, current, true);
	}

handleScroll函数定义在vue-router源码的1547 ~ 1587行：

	function handleScroll (
	  router,
	  to,
	  from,
	  isPop
	) {
	  // 还没有挂载app，return
	  if (!router.app) {
	    return
	  }
	  // 有设置behavior, return
	  var behavior = router.options.scrollBehavior;
	  if (!behavior) {
	    return
	  }
	
	  {
	    assert(typeof behavior === 'function', "scrollBehavior must be a function");
	  }
	
	  // wait until re-render finishes before scrolling
	  // 调用vue实例定义的nextTick
	  router.app.$nextTick(function () {
	    var position = getScrollPosition();//拿到位置，判断是否需要scroll
	    var shouldScroll = behavior(to, from, isPop ? position : null);
	    // 不需要滚动则直接返回
	    if (!shouldScroll) {
	      return
	    }
	    // 滚动到对应位置
	    if (typeof shouldScroll.then === 'function') {//支持promise
	      shouldScroll.then(function (shouldScroll) {
	        scrollToPosition((shouldScroll), position);
	      }).catch(function (err) {
	        {
	          assert(false, err.toString());
	        }
	      });
	    } else {
	      scrollToPosition(shouldScroll, position);//不支持promise的话
	    }
	  });
	}

可以看到，它接收四个参数，分别表示router实例、目标路由、路由来源、是否支持popState。 该函数在进行一些基本的校验后会调用vue的$nextTick方法注册一个异步函数。该异步函数首先调用getScrollPosition拿到当前`_Key`对应的页面的位置：

	var position = getScrollPosition();//拿到位置，判断是否需要scroll

getScrollPosition定义在源码的1599 ~ 1604行，主要就是根据当前`_Key`取出保存的位置：

	//拿位置
	function getScrollPosition () {
	  var key = getStateKey();//获取对应的位置
	  if (key) {
	    return positionStore[key]
	  }
	}

然后会执行实例化时用户传递的scrollBehavior函数，开篇已经提到过scrollBehavior函数的不同返回值及其含义。如果scrollBehavior返回falsy，则表明不需要滚动。否则调用scrollToPosition完成真正的滚动：

	// 滚动到对应位置
    if (typeof shouldScroll.then === 'function') {//支持promise
      shouldScroll.then(function (shouldScroll) {
        scrollToPosition((shouldScroll), position);
      }).catch(function (err) {
        {
          assert(false, err.toString());
        }
      });
    } else {
      scrollToPosition(shouldScroll, position);//不支持promise的话
    }

scrollToPosition定义在源码的1638 ~ 1656行，会根据实例化router时传递的scrollBehavior函数的返回值和拿到额position执行滚动：

	//滚动到对应位置
	//调用的实际上还是window.scrollTo
	function scrollToPosition (shouldScroll, position) {
	  var isObject = typeof shouldScroll === 'object';
	  if (isObject && typeof shouldScroll.selector === 'string') {
	    var el = document.querySelector(shouldScroll.selector);
	    if (el) {
	      var offset = shouldScroll.offset && typeof shouldScroll.offset === 'object' ? shouldScroll.offset : {};
	      offset = normalizeOffset(offset);
	      position = getElementPosition(el, offset);
	    } else if (isValidPosition(shouldScroll)) {
	      position = normalizePosition(shouldScroll);
	    }
	  } else if (isObject && isValidPosition(shouldScroll)) {
	    position = normalizePosition(shouldScroll);
	  }
	
	  if (position) {
	    window.scrollTo(position.x, position.y);
	  }
	}

scrollToPosition会根据传scrollBehavior函数的返回值(这里是shouldScroll对象)的内容区别对待。 如果shouldScroll拥有selector属性，则实现的是『滚动到锚点』的行为，当有selector属性时，会拿到对应的元素。它还会进一步根据是否有offset属性来得到最终的位置。offset需要用normalizeOffset函数进行规范化，normalizeOffset定义在源码的1627 ~ 1632行：

	function normalizeOffset (obj) {//归一化/规范化偏移量
	  return {
	    x: isNumber(obj.x) ? obj.x : 0,
	    y: isNumber(obj.y) ? obj.y : 0
	  }
	}
	
主要就是确保position是数值，调用的是isNumber方法，定义在1634 ~ 1636行：

	function isNumber (v) {//是否是数值
	  return typeof v === 'number'
	}

其实这里完全可以调用Number.isNumber方法，不用自己定义。

然后会依据锚点元素和offset调用getElementPosition拿到正确的position, getElementPosition定义在源码的1606 ~ 1614行：

	function getElementPosition (el, offset) {//获得元素的位置
	  var docEl = document.documentElement;
	  var docRect = docEl.getBoundingClientRect();//文档的区域
	  var elRect = el.getBoundingClientRect();//元素的区域
	  return {
	    x: elRect.left - docRect.left - offset.x,
	    y: elRect.top - docRect.top - offset.y
	  }
	}

而如果页面实际上没有selector属性指定的锚点元素时，会调用isValidPosition对shouldScroll对象进行验证，主要是验证x/y属性是不是Number， isValidPosition定义在源码的1616 ~ 1619行：

	function isValidPosition (obj) {//判断位置是否有效
	  return isNumber(obj.x) || isNumber(obj.y)
	}

如果验证通过还会对shouldScroll对象进行规范化，调用的是normalizePosition,定义在源码的1620 ~ 1625行：

	function normalizePosition (obj) {//归一化/规范化位置
	  return {
	    x: isNumber(obj.x) ? obj.x : window.pageXOffset,
	    y: isNumber(obj.y) ? obj.y : window.pageYOffset
	  }
	}

如果shouldScroll对象没有selector属性，也会执行“有selector属性但指定的锚点不存在”时一样的操作。当确定最终的position后会调用原生的scrollTo滚动到指定的位置。

至此，已经将开篇管方文档的那篇描述完全从源代码的角度解释了一遍。最后我们看看官方给出的例子来验证一下：

	Vue.use(VueRouter)
	
	const Home = { template: '<div>home</div>' }
	const Foo = { template: '<div>foo</div>' }
	const Bar = {
	  template: `
	    <div>
	      bar
	      <div style="height:1500px"></div>
	      <p id="anchor">Anchor</p>
	    </div>
	  `
	}
	
	// scrollBehavior:
	// - only available in html5 history mode
	// - defaults to no scroll behavior
	// - return false to prevent scroll
	const scrollBehavior = (to, from, savedPosition) => {
	  console.log(savedPosition)
	  if (savedPosition) {
	    // savedPosition is only available for popstate navigations.
	    return savedPosition
	  } else {
	    const position = {}
	    // new navigation.
	    // scroll to anchor by returning the selector
	    if (to.hash) {
	      position.selector = to.hash
	    }
	    // check if any matched route config has meta that requires scrolling to top
	    // console.log(to.matched)
	    if (to.matched.some(m => m.meta.scrollToTop)) {
	      // cords will be used if no selector is provided,
	      // or if the selector didn't match any element.
	      position.x = 0
	      position.y = 0
	    }
	    // if the returned position is falsy or an empty object,
	    // will retain current scroll position.
	    return position
	  }
	}
	
	const router = new VueRouter({
	  mode: 'history',
	  base: '/',
	  scrollBehavior,
	  routes: [
	    { path: '/', component: Home, meta: { scrollToTop: true }},
	    { path: '/foo', component: Foo },
	    { path: '/bar', component: Bar, meta: { scrollToTop: true }}
	  ]
	})
	
	new Vue({
	  router,
	  template: `
	    <div id="app">
	      <h1>Scroll Behavior</h1>
	      <ul>
	        <li><router-link to="/">/</router-link></li>
	        <li><router-link to="/foo">/foo</router-link></li>
	        <li><router-link to="/bar">/bar</router-link></li>
	        <li><router-link to="/bar#anchor">/bar#anchor</router-link></li>
	      </ul>
	      <router-view class="view"></router-view>
	    </div>
	  `
	}).$mount('#app')

我们主要看scrollBehavior

	const scrollBehavior = (to, from, savedPosition) => {
	  console.log(savedPosition)
	  if (savedPosition) {
	    // savedPosition is only available for popstate navigations.
	    return savedPosition
	  } else {
	    const position = {}
	    // new navigation.
	    // scroll to anchor by returning the selector
	    if (to.hash) {
	      position.selector = to.hash
	    }
	    // check if any matched route config has meta that requires scrolling to top
	    // console.log(to.matched)
	    if (to.matched.some(m => m.meta.scrollToTop)) {
	      // cords will be used if no selector is provided,
	      // or if the selector didn't match any element.
	      position.x = 0
	      position.y = 0
	    }
	    // if the returned position is falsy or an empty object,
	    // will retain current scroll position.
	    return position
	  }
	}

这里有三个路由，四个link。‘/’,‘/bar’路由都指定了滚动到顶部，‘/foo’没有。

当用户按前进/后退的时候，会执行：

	// savedPosition is only available for popstate navigations.
	return savedPosition

点击"/bar#anchor"链接时，position将含有selector，x，y三个属性：

![](/assets/2.png)

![](https://i.imgur.com/vGz5ukv.png)

因为页面有 `<p id="anchor">Anchor</p>`锚点，所以会执行vue-router源码scrollToPosition函数的如下分支：

	function scrollToPosition (shouldScroll, position) {
	  var isObject = typeof shouldScroll === 'object';
	  if (isObject && typeof shouldScroll.selector === 'string') {
	    var el = document.querySelector(shouldScroll.selector);
	    if (el) {
	      var offset = shouldScroll.offset && typeof shouldScroll.offset === 'object' ? shouldScroll.offset : {};
	      offset = normalizeOffset(offset);
	      position = getElementPosition(el, offset);

