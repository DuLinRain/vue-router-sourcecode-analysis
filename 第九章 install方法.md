# 第九章 install方法
vue-router也就是一个vue插件，所以它必然也有一个install方法用于插件的注册，vue-router的install方法定义在源码的526 ~ 572行，它主要是完成了钩子函数的混入以及router-view组件、router-link组件的注册，我们来具体看看：

	function install (Vue) {
	  if (install.installed && _Vue === Vue) { return }
	  install.installed = true;
	
	  _Vue = Vue;
	
	  var isDef = function (v) { return v !== undefined; };
	
	  var registerInstance = function (vm, callVal) {
	    var i = vm.$options._parentVnode;
	    if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
	      i(vm, callVal);
	    }
	  };
	
	  Vue.mixin({
	    beforeCreate: function beforeCreate () {
	      if (isDef(this.$options.router)) {
	        this._routerRoot = this;
	        this._router = this.$options.router;
	        this._router.init(this);
	        Vue.util.defineReactive(this, '_route', this._router.history.current);
	      } else {
	        this._routerRoot = (this.$parent && this.$parent._routerRoot) || this;
	      }
	      registerInstance(this, this);
	    },
	    destroyed: function destroyed () {
	      registerInstance(this);
	    }
	  });
	
	  Object.defineProperty(Vue.prototype, '$router', {
	    get: function get () { return this._routerRoot._router }
	  });
	
	  Object.defineProperty(Vue.prototype, '$route', {
	    get: function get () { return this._routerRoot._route }
	  });
	
	  Vue.component('router-view', View);
	  Vue.component('router-link', Link);
	
	  var strats = Vue.config.optionMergeStrategies;
	  // use the same hook merging strategy for route hooks
	  strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created;
	}

最开始，为了保证正确且只安装一次，会检查Vue是否引入以及Vue-Router已经被安装过：

	if (install.installed && _Vue === Vue) { return }
		  install.installed = true;

随后会混入vue-router插件的beforeCreate和destroyed钩子函数：

	Vue.mixin({
	    beforeCreate: function beforeCreate () {
	      if (isDef(this.$options.router)) {
	        this._routerRoot = this;
	        this._router = this.$options.router;
	        this._router.init(this);
	        Vue.util.defineReactive(this, '_route', this._router.history.current);
	      } else {
	        this._routerRoot = (this.$parent && this.$parent._routerRoot) || this;
	      }
	      registerInstance(this, this);
	    },
	    destroyed: function destroyed () {
	      registerInstance(this);
	    }
	  });

在beforeCreate钩子函数中会将vue实例挂载在vue实例自身的\_routerRoot属性上，将router实例挂在在vue实例自身的\_router属性上，接着调用router实例的init方法。init方法我们在VueRouter类中已经介绍过了。 并且还会定义\_router属性为响应式的。

随后会注册router-view、router-link组件，采用的也是全局注册到Vue上：

	Vue.component('router-view', View);
	Vue.component('router-link', Link);

最后定义了beforeRouteEnter、beforeRouteLeave、beforeRouteUpdate守卫的合并策略，他们的合并策略和vue的created钩子函数的合并策略是一样的。

最后看一个例子的vue实例：

![](/assets/vue-router10.png)

![](https://i.imgur.com/lJeW2WX.png)