# 第六章 HashHistory类
### 6.1 HashHistory类源码

HashHistory类定义在vue-router源码的2231 ~ 2314行，和HTML5History模式实现的方式以及接口基本一样，采用的也是经典的IIFE(立即调用函数表达式)方式实现的：

	var HashHistory = (function (History$$1) {
	  function HashHistory (router, base, fallback) {
	    History$$1.call(this, router, base);//调用History类的构造函数
	    // check history fallback deeplinking
	    if (fallback && checkFallback(this.base)) {
	      return
	    }
	    ensureSlash();//确保反斜线
	  }
	  //经典的类继承实现方式
	  if ( History$$1 ) HashHistory.__proto__ = History$$1;
	  HashHistory.prototype = Object.create( History$$1 && History$$1.prototype );
	  HashHistory.prototype.constructor = HashHistory;
	
	  // this is delayed until the app mounts
	  // to avoid the hashchange listener being fired too early
	  HashHistory.prototype.setupListeners = function setupListeners () {
	    var this$1 = this;
	
	    var router = this.router;
	    var expectScroll = router.options.scrollBehavior;
	    var supportsScroll = supportsPushState && expectScroll;
	
	    if (supportsScroll) {
	      setupScroll();
	    }
	    //如果支持popstate则使用它，否则使用hashchange
	    //如果支持popstate，则监听popstate，否则监听hashchange
	    window.addEventListener(supportsPushState ? 'popstate' : 'hashchange', function () {
	      var current = this$1.current;
	      if (!ensureSlash()) {
	        return
	      }
	      this$1.transitionTo(getHash(), function (route) {
	        if (supportsScroll) {
	          handleScroll(this$1.router, route, current, true);
	        }
	        if (!supportsPushState) {
	          replaceHash(route.fullPath);
	        }
	      });
	    });
	  };
	  //HashHistory的push方法，调用的是pushHash方法
	  HashHistory.prototype.push = function push (location, onComplete, onAbort) {
	    var this$1 = this;
	
	    var ref = this;
	    var fromRoute = ref.current;//当前路由
	    this.transitionTo(location, function (route) {
	      pushHash(route.fullPath);
	      handleScroll(this$1.router, route, fromRoute, false);
	      onComplete && onComplete(route);
	    }, onAbort);
	  };
	  //HashHistory的replace方法,调用的是replacehHash方法
	  HashHistory.prototype.replace = function replace (location, onComplete, onAbort) {
	    var this$1 = this;
	
	    var ref = this;
	    var fromRoute = ref.current;
	    this.transitionTo(location, function (route) {
	      replaceHash(route.fullPath);
	      handleScroll(this$1.router, route, fromRoute, false);
	      onComplete && onComplete(route);
	    }, onAbort);
	  };
	  //HashHistory的go方法，调用的是window.history.go方法
	  HashHistory.prototype.go = function go (n) {
	    window.history.go(n);
	  };
	  //HashHistory的ensureURL方法
	  //参数决定是push还是replace
	  HashHistory.prototype.ensureURL = function ensureURL (push) {
	    //当前路径的完全路径
	    var current = this.current.fullPath;
	    if (getHash() !== current) {
	      push ? pushHash(current) : replaceHash(current);
	    }
	  };
	  //HashHistory的getCurrentLocation方法，调用的是getHash方法
	  //拿url中#符号后面的东西
	  HashHistory.prototype.getCurrentLocation = function getCurrentLocation () {
	    return getHash()
	  };
	
	  return HashHistory;
	}(History));


该类继承自History类，继承的方式也是经典的原型继承方式，我们来大体看一下：

该函数的构造函数中首先借用父类的构造函数（2233行）：

	History$$1.call(this, router, base);//调用History类的构造函数

然后实现原型继承（2241 ~ 2243行）：

	  if ( History$$1 ) HashHistory.__proto__ = History$$1;
	  HashHistory.prototype = Object.create( History$$1 && History$$1.prototype );
	  HashHistory.prototype.constructor = HashHistory;

这样就完成了从History类继承。

构造函数会先检查是否支持回退(fallback)，如果支持回退，并且base是以/#开头的话，不会调用ensureSlash强制path以/开头。

ensureSlash函数定义在源码的2326 ~ 2333行，它会先调用getHash拿到hash，然后确保以/开头：

	function ensureSlash () {
	  var path = getHash();
	  if (path.charAt(0) === '/') {
	    return true
	  }
	  replaceHash('/' + path);
	  return false
	}

getHash函数定义在2335 ~ 2341行，它直接截取#后面的内容返回：

	//手动拿hash，而不是用window.location.hash
	//为了确保跨浏览器支持
	function getHash () {
	  // We can't use window.location.hash here because it's not
	  // consistent across browsers - Firefox will pre-decode it!
	  var href = window.location.href;
	  var index = href.indexOf('#');
	  return index === -1 ? '' : href.slice(index + 1)
	}

HashHistory类定义了一些与浏览器操作有关的原型函数，这些函数的名称与HTML5History几乎完全一样，我们分别来看一下。

#### 6.1.1 原型函数

##### 6.1.1.1 HashHistory.prototype.setupListeners

	  // this is delayed until the app mounts
	  // to avoid the hashchange listener being fired too early
	  HashHistory.prototype.setupListeners = function setupListeners () {
	    var this$1 = this;
	
	    var router = this.router;
	    var expectScroll = router.options.scrollBehavior;
	    var supportsScroll = supportsPushState && expectScroll;
	
	    if (supportsScroll) {
	      setupScroll();
	    }
	    //如果支持popstate则使用它，否则使用hashchange
	    //如果支持popstate，则监听popstate，否则监听hashchange
	    window.addEventListener(supportsPushState ? 'popstate' : 'hashchange', function () {
	      var current = this$1.current;
	      if (!ensureSlash()) {
	        return
	      }
	      this$1.transitionTo(getHash(), function (route) {
	        if (supportsScroll) {
	          handleScroll(this$1.router, route, current, true);
	        }
	        if (!supportsPushState) {
	          replaceHash(route.fullPath);
	        }
	      });
	    });
	  };

setupListeners就和HTML5History的构造函数的道理是一样的，主要就是处理滚动以及监听响应的事件(popstate/hashchange)。setupListeners最终会在VueRouter的init函数中被调用。

##### 6.1.1.2 HashHistory.prototype.push
HashHistory.prototype.push方法定义在源码的2274 ~ 2284行，该方法用于HashHistory模式下将当新路由加入到history中。从代码来看它在调用transitionTo方法的成功回调中会调用handleScroll方法，所以hash模式应该也是支持滚动的：

	  //HashHistory的replace方法,调用的是replacehHash方法
	  HashHistory.prototype.replace = function replace (location, onComplete, onAbort) {
	    var this$1 = this;
	
	    var ref = this;
	    var fromRoute = ref.current;
	    this.transitionTo(location, function (route) {
	      replaceHash(route.fullPath);
	      handleScroll(this$1.router, route, fromRoute, false);
	      onComplete && onComplete(route);
	    }, onAbort);
	  };

##### 6.1.1.3 HashHistory.prototype.replace
HashHistory.prototype.replace方法定义在源码的2286 ~ 2296行，该方法用于HashHistory模式下将将新路由替换history中当前路由。从代码来看它在调用transitionTo方法的成功回调中会调用handleScroll方法，所以hash模式应该也是支持滚动的：

	  //HashHistory的replace方法,调用的是replacehHash方法
	  HashHistory.prototype.replace = function replace (location, onComplete, onAbort) {
	    var this$1 = this;
	
	    var ref = this;
	    var fromRoute = ref.current;
	    this.transitionTo(location, function (route) {
	      replaceHash(route.fullPath);
	      handleScroll(this$1.router, route, fromRoute, false);
	      onComplete && onComplete(route);
	    }, onAbort);
	  };

##### HashHistory.prototype.go
HashHistory.prototype.go方法定义在2298 ~ 2300行，它实际上调用的就是原生的window.history.go(n)来实现的：

	  //HashHistory的go方法，调用的是window.history.go方法
	  HashHistory.prototype.go = function go (n) {
	    window.history.go(n);
	  };

##### HashHistory.prototype.ensureURL
HashHistory.prototype.ensureURL定义在vue-router源码的2302 ~ 2307 行，它的用处和HTML5History.prototype.ensureURL，只不过它是比较二者的hash是不是一致的，并且当不一致时也是调用的pushHash/replaceHash来新增/替换。

	  //参数决定是push还是replace
	  HashHistory.prototype.ensureURL = function ensureURL (push) {
	    //当前路径的完全路径
	    var current = this.current.fullPath;
	    if (getHash() !== current) {
	      push ? pushHash(current) : replaceHash(current);
	    }
	  };

getHash方法定义在源码的2235 ~ 2341行，是通过href截取的hash，而不是直接window.location.hash：

	//手动拿hash，而不是用window.location.hash
	//为了确保跨浏览器支持
	function getHash () {
		  // We can't use window.location.hash here because it's not
		  // consistent across browsers - Firefox will pre-decode it!
		  var href = window.location.href;
		  var index = href.indexOf('#');
		  return index === -1 ? '' : href.slice(index + 1)
	}

##### HashHistory.prototype.getCurrentLocation
HashHistory.prototype.getCurrentLocation定义在源码的2309 ~ 2311行，用于拿到当前的hash，调用的实际上是getHash，这个在前面已经讲过了：

	HashHistory.prototype.getCurrentLocation = function getCurrentLocation () {
	    return getHash()
	};

##