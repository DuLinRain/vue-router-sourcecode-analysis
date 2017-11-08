# 第七章 AbstractHistory类
### 7.1 AbstractHistory类源码分析

AbstractHistory类定义在vue-router源码的2369 ~ 2423行，采用的也是经典的IIFE(立即调用函数表达式)方式实现的：

	//AbstractHistory类
	var AbstractHistory = (function (History$$1) {
	  function AbstractHistory (router, base) {
	    //借用父类的构造函数
	    History$$1.call(this, router, base);
	    this.stack = [];
	    this.index = -1;
	  }
	  //经典的类继承
	  if ( History$$1 ) AbstractHistory.__proto__ = History$$1;
	  AbstractHistory.prototype = Object.create( History$$1 && History$$1.prototype );
	  AbstractHistory.prototype.constructor = AbstractHistory;
	  //实现响应的push方法
	  AbstractHistory.prototype.push = function push (location, onComplete, onAbort) {
	    var this$1 = this;
	
	    this.transitionTo(location, function (route) {
	      this$1.stack = this$1.stack.slice(0, this$1.index + 1).concat(route);
	      this$1.index++;
	      onComplete && onComplete(route);
	    }, onAbort);
	  };
	  //实现响应的replace方法
	  AbstractHistory.prototype.replace = function replace (location, onComplete, onAbort) {
	    var this$1 = this;
	
	    this.transitionTo(location, function (route) {
	      this$1.stack = this$1.stack.slice(0, this$1.index).concat(route);
	      onComplete && onComplete(route);
	    }, onAbort);
	  };
	  //实现响应的go方法
	  AbstractHistory.prototype.go = function go (n) {
	    var this$1 = this;
	
	    var targetIndex = this.index + n;
	    if (targetIndex < 0 || targetIndex >= this.stack.length) {
	      return
	    }
	    var route = this.stack[targetIndex];
	    this.confirmTransition(route, function () {
	      this$1.index = targetIndex;
	      this$1.updateRoute(route);
	    });
	  };
	  //实现响应的getCurrentLocation方法
	  AbstractHistory.prototype.getCurrentLocation = function getCurrentLocation () {
	    var current = this.stack[this.stack.length - 1];
	    return current ? current.fullPath : '/'
	  };
	  //实现响应的ensureURL方法
	  AbstractHistory.prototype.ensureURL = function ensureURL () {
	    // noop
	  };
	
	  return AbstractHistory;
	}(History));

它于HTML5History和HashHistory的模式基本上是一样的，但是它不支持滚动行为。另外，它采用的是stack数组来存储每次的路由历史，这个也是与前面两种类型有区别的。

该类继承自History类，继承的方式也是经典的原型继承方式，我们来大体看一下：

该函数的构造函数中首先借用父类的构造函数（2371行）：

	//借用父类的构造函数
	History$$1.call(this, router, base);

然后实现原型继承（2376 ~ 2379）：

	  //经典的类继承
	  if ( History$$1 ) AbstractHistory.__proto__ = History$$1;
	  AbstractHistory.prototype = Object.create( History$$1 && History$$1.prototype );
	  AbstractHistory.prototype.constructor = AbstractHistory;

这样就完成了从History类继承。

HashHistory类定义了一些与浏览器操作有关的原型函数，这些函数的名称与HTML5History、HashHistory几乎完全一样，我们分别来看一下。

#### 7.1.1 原型函数
##### 7.1.1.1 AbstractHistory.prototype.push
AbstractHistory.prototype.push定义在源码的2380 ~ 2388行，AbstractHistory模式下push的实现与前两种有所差异，因为这种模式下的history是通过statck数组记录的，所以push就是把当前路由加入到数组中，然后执行transitionTo方法：

	//实现响应的push方法
	AbstractHistory.prototype.push = function push (location, onComplete, onAbort) {
		var this$1 = this;
		
		this.transitionTo(location, function (route) {
		  this$1.stack = this$1.stack.slice(0, this$1.index + 1).concat(route);
		  this$1.index++;
		  onComplete && onComplete(route);
		}, onAbort);
	};

##### 7.1.1.2 AbstractHistory.prototype.replace
AbstractHistory.prototype.replace定义在源码的2390 ~ 2397行，同AbstractHistory.prototype.push类似，只不过这里的slice不一样(`this$1.stack.slice(0, this$1.index)`)：

	  //实现响应的replace方法
	  AbstractHistory.prototype.replace = function replace (location, onComplete, onAbort) {
	    var this$1 = this;
	
	    this.transitionTo(location, function (route) {
	      this$1.stack = this$1.stack.slice(0, this$1.index).concat(route);
	      onComplete && onComplete(route);
	    }, onAbort);
	  };


##### 7.1.1.3 AbstractHistory.prototype.go
AbstractHistory.prototype.go定义在源码的2399 ~ 2411行，前进/后退的时候就是取对应的索引，如果索引越界则会直接返回：

	  //实现响应的go方法
	  AbstractHistory.prototype.go = function go (n) {
	    var this$1 = this;
	
	    var targetIndex = this.index + n;
	    if (targetIndex < 0 || targetIndex >= this.stack.length) {
	      return
	    }
	    var route = this.stack[targetIndex];
	    this.confirmTransition(route, function () {
	      this$1.index = targetIndex;
	      this$1.updateRoute(route);
	    });
	  };

##### 7.1.1.4 AbstractHistory.prototype.ensureURL
AbstractHistory.prototype.ensureURL是一个空函数：

	//实现响应的ensureURL方法
	AbstractHistory.prototype.ensureURL = function ensureURL () {
		// noop
	};

##### 7.1.1.5 AbstractHistory.prototype.getCurrentLocation
AbstractHistory.prototype.getCurrentLocation定义在源码的2413 ~ 2416行，直接取stack中存储的最后一个路由：

	//实现响应的getCurrentLocation方法
	AbstractHistory.prototype.getCurrentLocation = function getCurrentLocation () {
	    var current = this.stack[this.stack.length - 1];
	    return current ? current.fullPath : '/'
	};