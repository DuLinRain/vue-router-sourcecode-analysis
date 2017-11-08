# 第二章 route-view组件

### 2.1 route-view组件定义在vue-router源码的30 ~ 381行，我们来看一看它的实现：

	var View = {
	  name: 'router-view',
	  functional: true,
	  props: {
	    name: {
	      type: String,
	      default: 'default'
	    }
	  },
	  render: function render (_, ref) {
	    var props = ref.props;
	    var children = ref.children;
	    var parent = ref.parent;
	    var data = ref.data;
	
	    data.routerView = true;
	
	    // directly use parent context's createElement() function
	    // so that components rendered by router-view can resolve named slots
	    var h = parent.$createElement;
	    var name = props.name;
	    var route = parent.$route;
	    var cache = parent._routerViewCache || (parent._routerViewCache = {});
	
	    // determine current view depth, also check to see if the tree
	    // has been toggled inactive but kept-alive.
	    var depth = 0;
	    var inactive = false;
	    while (parent && parent._routerRoot !== parent) {
	      if (parent.$vnode && parent.$vnode.data.routerView) {
	        depth++;
	      }
	      if (parent._inactive) {
	        inactive = true;
	      }
	      parent = parent.$parent;
	    }
	    data.routerViewDepth = depth;
	
	    // render previous view if the tree is inactive and kept-alive
	    if (inactive) {
	      return h(cache[name], data, children)
	    }
	
	    var matched = route.matched[depth];
	    // render empty node if no matched route
	    if (!matched) {
	      cache[name] = null;
	      return h()
	    }
	
	    var component = cache[name] = matched.components[name];
	
	    // attach instance registration hook
	    // this will be called in the instance's injected lifecycle hooks
	    data.registerRouteInstance = function (vm, val) {
	      // val could be undefined for unregistration
	      var current = matched.instances[name];
	      if (
	        (val && current !== vm) ||
	        (!val && current === vm)
	      ) {
	        matched.instances[name] = val;
	      }
	    }
	
	    // also register instance in prepatch hook
	    // in case the same component instance is reused across different routes
	    ;(data.hook || (data.hook = {})).prepatch = function (_, vnode) {
	      matched.instances[name] = vnode.componentInstance;
	    };
	
	    // resolve props
	    var propsToPass = data.props = resolveProps(route, matched.props && matched.props[name]);
	    if (propsToPass) {
	      // clone to prevent mutation
	      propsToPass = data.props = extend({}, propsToPass);
	      // pass non-declared props as attrs
	      var attrs = data.attrs = data.attrs || {};
	      for (var key in propsToPass) {
	        if (!component.props || !(key in component.props)) {
	          attrs[key] = propsToPass[key];
	          delete propsToPass[key];
	        }
	      }
	    }
	
	    return h(component, data, children)
	  }
	};

`<router-view> `组件是一个 functional 组件，渲染路径匹配到的视图组件。`<router-view>` 渲染的组件还可以内嵌自己的 `<router-view>`，根据嵌套路径，渲染嵌套组件。 如果 `<router-view>`设置了名称，则会渲染对应的路由配置中 components 下的相应组件。函数组件可能大家不常用，但是官方文档对函数组件有比较详细的描述：

![](/assets/2-1.png)

![](https://i.imgur.com/WhVBMr2.png)

![](/assets/2-2.png)

![](https://i.imgur.com/yw2Ku0l.png)

![](/assets/2-3.png)

![](https://i.imgur.com/bpEpmSo.png)

router-view组件主要的做的就是通过调用createElement方法来渲染我们点击router-link后匹配到的组件：

	return h(component, data, children)

这里面在解析属性的时候调用了resolveProps方法，该方法根据当前URL生成的路由和匹配到的注册的路由的props来解析，resolveProps定义在vue-router源码的121 ~ 140行：

	function resolveProps (route, config) {
	  switch (typeof config) {
	    case 'undefined':
	      return
	    case 'object':
	      return config
	    case 'function':
	      return config(route)
	    case 'boolean':
	      return config ? route.params : undefined
	    default:
	      {
	        warn(
	          false,
	          "props in \"" + (route.path) + "\" is a " + (typeof config) + ", " +
	          "expecting an object, function or boolean."
	        );
	      }
	  }
	}

resolveProps函数根据第二个参数的类型来确定返回何种值。当返回的类型不为falsy的时候，会对解析到的props进行浅拷贝，浅拷贝采用的是extend方法，定义在源码的142 ~ 147行：

	function extend (to, from) {
	  for (var key in from) {
	    to[key] = from[key];
	  }
	  return to
	}