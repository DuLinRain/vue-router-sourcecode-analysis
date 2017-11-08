# 第八章 VueRouter类

### 8.1 VueRouter类源码分析

VueRouter类定义在vue-router源码的2427 ~ 2463行，我们来看一下它的实现。

#### 8.1.1 成员属性
VueRouter的构造函数接收options作为参数，共定义了10个成员属性，分别是：

- app
- apps
- options。存储实例化时传递的options参数。
- beforeHooks。前置钩子函数。
- resolveHooks。解析钩子函数。
- afterHooks。后置钩子函数。
- matcher。是一个对象，包含match和addRoutes两个函数。
- fallback。类型: boolean。当浏览器不支持 history.pushState 控制路由是否应该回退到 hash 模式。默认值为 true。在 IE9 中，设置为 false 会使得每个 router-link 导航都触发整页刷新。它可用于工作在 IE9 下的服务端渲染应用，因为一个 hash 模式的 URL 并不支持服务端渲染
- mode。配置路由模式。
#
![](/assets/vue-router1.png)

![](https://i.imgur.com/EauyRTa.png)
#
- history。根据不同的mode来决定实例化什么样的history。hash模式实例化HashHistory类，history模式实例化Html5History类，abstract模式实例化AbstractHistory类。

构造函数的源码如下：

	var VueRouter = function VueRouter (options) {
	  if ( options === void 0 ) options = {};//没传递options时，赋默认值
	
	  this.app = null;//类型: Vue instance,配置了 router 的 Vue 根实例
	  this.apps = [];
	  this.options = options;//router实例上挂载options
	  this.beforeHooks = [];//全局前置守卫钩子
	  this.resolveHooks = [];//解析钩子
	  this.afterHooks = [];//后置钩子
	  this.matcher = createMatcher(options.routes || [], this);//根据routes和router实例来创建matcher
	
	  var mode = options.mode || 'hash';//模式
	  this.fallback = mode === 'history' && !supportsPushState && options.fallback !== false;
	  if (this.fallback) {
	    mode = 'hash';
	  }
	  if (!inBrowser) {//非浏览器模式下采用abstract模式
	    mode = 'abstract';
	  }
	  this.mode = mode;
	  //根据模式确定history属性
	  switch (mode) {
	    case 'history':
	      this.history = new HTML5History(this, options.base);
	      break
	    case 'hash':
	      this.history = new HashHistory(this, options.base, this.fallback);
	      break
	    case 'abstract':
	      this.history = new AbstractHistory(this, options.base);
	      break
	    default:
	      {
	        assert(false, ("invalid mode: " + mode));
	      }
	  }
	};

在实例化router的时候会给matcher属性赋值，调用的是createMatcher方法，该方法的实现比较复杂，我们接下来详细看看它的实现。

#### 8.1.2 createMatcher方法
createMatcher方法定义在vue-router源码的1338 ~ 1501行，它的作用是根据router实例以及实例化时传递的options中的routes来生成一个matcher对象，这个对象包含addRoutes和match两个方法，addRoutes方法用于向pathList、pathMap、nameMap中增加路由，其实就是更新pathList、pathMap、nameMap。pathList存储的是所有路由(包括子路由)的path(子路由的path经过了规范化处理，也就是说会有链路区分)，pathMap是一个对象，存储的是path->record对，nameMap也是一个对象，存储的是name->record对。下面具体看一下它的实现：

	//生成匹配者
	function createMatcher (
	  routes,//routes数组
	  router//router实例
	) {
	  //生成pathList、pathMap、nameMap映射表
	  var ref = createRouteMap(routes);
	  var pathList = ref.pathList;
	  var pathMap = ref.pathMap;
	  var nameMap = ref.nameMap;
	  //定义一个addRoutes方法，实际就是把routes加入到已有的routes从而更新了Map和List
	  function addRoutes (routes) {
	    createRouteMap(routes, pathList, pathMap, nameMap);
	  }
	  //返回一个匹配的路由
	  function match (
	    raw,
	    currentRoute,
	    redirectedFrom
	  ) {
	    var location = normalizeLocation(raw, currentRoute, false, router);
	    var name = location.name;
	
	    if (name) {
	      var record = nameMap[name];
	      {
	        warn(record, ("Route with name '" + name + "' does not exist"));
	      }
	      if (!record) { return _createRoute(null, location) }
	      var paramNames = record.regex.keys
	        .filter(function (key) { return !key.optional; })
	        .map(function (key) { return key.name; });
	
	      if (typeof location.params !== 'object') {
	        location.params = {};
	      }
	
	      if (currentRoute && typeof currentRoute.params === 'object') {
	        for (var key in currentRoute.params) {
	          if (!(key in location.params) && paramNames.indexOf(key) > -1) {
	            location.params[key] = currentRoute.params[key];
	          }
	        }
	      }
	
	      if (record) {
	        location.path = fillParams(record.path, location.params, ("named route \"" + name + "\""));
	        return _createRoute(record, location, redirectedFrom)
	      }
	    } else if (location.path) {
	      location.params = {};
	      for (var i = 0; i < pathList.length; i++) {
	        var path = pathList[i];
	        var record$1 = pathMap[path];
	        if (matchRoute(record$1.regex, location.path, location.params)) {
	          return _createRoute(record$1, location, redirectedFrom)
	        }
	      }
	    }
	    // no match
	    return _createRoute(null, location)
	  }
	  //返回的也是一个路由
	  function redirect (
	    record,
	    location
	  ) {
	    var originalRedirect = record.redirect;
	    var redirect = typeof originalRedirect === 'function'
	        ? originalRedirect(createRoute(record, location, null, router))
	        : originalRedirect;
	
	    if (typeof redirect === 'string') {
	      redirect = { path: redirect };
	    }
	
	    if (!redirect || typeof redirect !== 'object') {
	      {
	        warn(
	          false, ("invalid redirect option: " + (JSON.stringify(redirect)))
	        );
	      }
	      return _createRoute(null, location)
	    }
	
	    var re = redirect;
	    var name = re.name;
	    var path = re.path;
	    var query = location.query;
	    var hash = location.hash;
	    var params = location.params;
	    query = re.hasOwnProperty('query') ? re.query : query;
	    hash = re.hasOwnProperty('hash') ? re.hash : hash;
	    params = re.hasOwnProperty('params') ? re.params : params;
	
	    if (name) {
	      // resolved named direct
	      var targetRecord = nameMap[name];
	      {
	        assert(targetRecord, ("redirect failed: named route \"" + name + "\" not found."));
	      }
	      return match({
	        _normalized: true,
	        name: name,
	        query: query,
	        hash: hash,
	        params: params
	      }, undefined, location)
	    } else if (path) {
	      // 1. resolve relative redirect
	      var rawPath = resolveRecordPath(path, record);
	      // 2. resolve params
	      var resolvedPath = fillParams(rawPath, params, ("redirect route with path \"" + rawPath + "\""));
	      // 3. rematch with existing query and hash
	      return match({
	        _normalized: true,
	        path: resolvedPath,
	        query: query,
	        hash: hash
	      }, undefined, location)
	    } else {
	      {
	        warn(false, ("invalid redirect option: " + (JSON.stringify(redirect))));
	      }
	      return _createRoute(null, location)
	    }
	  }
	
	  function alias (
	    record,
	    location,
	    matchAs
	  ) {
	    var aliasedPath = fillParams(matchAs, location.params, ("aliased route with path \"" + matchAs + "\""));
	    var aliasedMatch = match({
	      _normalized: true,
	      path: aliasedPath
	    });
	    if (aliasedMatch) {
	      var matched = aliasedMatch.matched;
	      var aliasedRecord = matched[matched.length - 1];
	      location.params = aliasedMatch.params;
	      return _createRoute(aliasedRecord, location)
	    }
	    return _createRoute(null, location)
	  }
	
	  function _createRoute (
	    record,
	    location,
	    redirectedFrom
	  ) {
	    if (record && record.redirect) {
	      return redirect(record, redirectedFrom || location)
	    }
	    if (record && record.matchAs) {
	      return alias(record, location, record.matchAs)
	    }
	    return createRoute(record, location, redirectedFrom, router)
	  }
	
	  return {
	    match: match,//该方法用于返回一个匹配的路由
	    addRoutes: addRoutes//该方法用于新增路由映射
	  }
	}

可以看到，它首先调用的是createRouteMap方法，根据当前的routes来得到pathList、pathMap、nameMap。然后定义了局部函数addRoutes和match，将他们作为一个对象的属性返回。

我们来看看createRouteMap的实现。它定义在vue-router源码的1108 ~ 1139行：

	/*  */
	/*
	 * 根据routes构造pathList、pathMap、nameMap映射
	 * 核心是addRouteRecord函数
	 */
	function createRouteMap (
	  routes,
	  oldPathList,
	  oldPathMap,
	  oldNameMap
	) {
	  // the path list is used to control path matching priority
	  var pathList = oldPathList || [];
	  // $flow-disable-line
	  var pathMap = oldPathMap || Object.create(null);
	  // $flow-disable-line
	  var nameMap = oldNameMap || Object.create(null);
	  //遍历加上record
	  routes.forEach(function (route) {
	    addRouteRecord(pathList, pathMap, nameMap, route);
	  });
	
	  // ensure wildcard routes are always at the end
	  // 确保通配符路由总是在最后
	  for (var i = 0, l = pathList.length; i < l; i++) {
	    if (pathList[i] === '*') {
	      pathList.push(pathList.splice(i, 1)[0]);
	      l--;
	      i--;
	    }
	  }
	
	  return {
	    pathList: pathList,
	    pathMap: pathMap,
	    nameMap: nameMap
	  }
	}

它接收四个参数。你可能注意到前面我们调用它时只传了第一个参数。那是因为最开始我们用options.routes去初始化时后三个参数肯定是不存在的，我们只用一个参数的目的就是为了初始化后三个参数pathList、pathMap、nameMap。 回过头来分析代码，首先会判断有没有后三个参数，如果没有的话就初始化为[]、{}、{}。 然后会遍历routes，调用addRouteRecord函数来实现真正的初始化/更新pathList、pathMap、nameMap。然后会调整pathList的顺序，确保通配符路由总是在列表最后。最后将pathList、pathMap、nameMap作为一个对象的属性返回。

前面说了初始化/更新pathList、pathMap、nameMap实际上还是调用的addRouteRecord来实现的。它才是真正实现pathList、pathMap、nameMap初始化/更新的函数，它定义在vue-router的1141 ~ 1250行，我们来看看它的实现：

	/*
	 * 根据route、parent、matchAs
	 * 构造出pathList、pathMap、nameMap
	 */
	function addRouteRecord (
	  pathList,
	  pathMap,
	  nameMap,
	  route,
	  parent,
	  matchAs
	) {
	  var path = route.path;
	  var name = route.name;
	  {
	    assert(path != null, "\"path\" is required in a route configuration.");
	    assert(
	      typeof route.component !== 'string',
	      "route config \"component\" for path: " + (String(path || name)) + " cannot be a " +
	      "string id. Use an actual component instead."
	    );
	  }
	
	  var pathToRegexpOptions = route.pathToRegexpOptions || {};
	  //规范化path
	  var normalizedPath = normalizePath(
	    path,
	    parent,
	    pathToRegexpOptions.strict
	  );
	
	  if (typeof route.caseSensitive === 'boolean') {
	    pathToRegexpOptions.sensitive = route.caseSensitive;
	  }
	  //record对象
	  var record = {
	    path: normalizedPath,
	    regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
	    components: route.components || { default: route.component },
	    instances: {},
	    name: name,
	    parent: parent,
	    matchAs: matchAs,
	    redirect: route.redirect,
	    beforeEnter: route.beforeEnter,
	    meta: route.meta || {},
	    props: route.props == null
	      ? {}
	      : route.components
	        ? route.props
	        : { default: route.props }
	  };
	  //如果有孩子
	  if (route.children) {
	    // Warn if route is named, does not redirect and has a default child route.
	    // If users navigate to this route by name, the default child will
	    // not be rendered (GH Issue #629)
	    {
	      if (route.name && !route.redirect && route.children.some(function (child) { return /^\/?$/.test(child.path); })) {
	        warn(
	          false,
	          "Named Route '" + (route.name) + "' has a default child route. " +
	          "When navigating to this named route (:to=\"{name: '" + (route.name) + "'\"), " +
	          "the default child route will not be rendered. Remove the name from " +
	          "this route and use the name of the default child route for named " +
	          "links instead."
	        );
	      }
	    }
	    route.children.forEach(function (child) {
	      var childMatchAs = matchAs
	        ? cleanPath((matchAs + "/" + (child.path)))
	        : undefined;
	        //递归调用
	      addRouteRecord(pathList, pathMap, nameMap, child, record, childMatchAs);
	    });
	  }
	  //设置了别名
	  if (route.alias !== undefined) {
	    var aliases = Array.isArray(route.alias)
	      ? route.alias
	      : [route.alias];
	
	    aliases.forEach(function (alias) {
	      var aliasRoute = {
	        path: alias,
	        children: route.children
	      };
	      //递归调用
	      addRouteRecord(
	        pathList,
	        pathMap,
	        nameMap,
	        aliasRoute,
	        parent,
	        record.path || '/' // matchAs
	      );
	    });
	  }
	  //是新的，跟新pathList、pathMap
	  //pathList里面存的是路径
	  //pathMap里面存的是record,路径为key
	  //
	  if (!pathMap[record.path]) {
	    pathList.push(record.path);
	    pathMap[record.path] = record;
	  }
	
	  if (name) {
	    //nameMap里面存的是record,路径为key
	    if (!nameMap[name]) {
	      nameMap[name] = record;
	    } else if ("development" !== 'production' && !matchAs) {
	      warn(
	        false,
	        "Duplicate named routes definition: " +
	        "{ name: \"" + name + "\", path: \"" + (record.path) + "\" }"
	      );
	    }
	  }
	}

addRouteRecord函数接收6个参数，实际上后面两个参数是在当route有children的时候做递归调用的。先过一下代码：

首先会拿到route上定义的path和name和pathToRegexpOptions：

	var path = route.path;
    var name = route.name;
	var pathToRegexpOptions = route.pathToRegexpOptions || {};

然后会对路径进行规范化，这个主要是考虑到route可能有children，而pathList、pathMap都是用path来区分的record，因此需要对路径处理。

接下来就是根据route信息来创建record对象了：

	//record对象
	  var record = {
	    path: normalizedPath,
	    regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
	    components: route.components || { default: route.component },
	    instances: {},
	    name: name,
	    parent: parent,
	    matchAs: matchAs,
	    redirect: route.redirect,
	    beforeEnter: route.beforeEnter,
	    meta: route.meta || {},
	    props: route.props == null
	      ? {}
	      : route.components
	        ? route.props
	        : { default: route.props }
	  };

然后判断route有没有children，如果有的话则递归调用addRouteRecord更新pathList、pathMap、nameMap：

	  //如果有孩子
	  if (route.children) {
	    // Warn if route is named, does not redirect and has a default child route.
	    // If users navigate to this route by name, the default child will
	    // not be rendered (GH Issue #629)
	    {
	      if (route.name && !route.redirect && route.children.some(function (child) { return /^\/?$/.test(child.path); })) {
	        warn(
	          false,
	          "Named Route '" + (route.name) + "' has a default child route. " +
	          "When navigating to this named route (:to=\"{name: '" + (route.name) + "'\"), " +
	          "the default child route will not be rendered. Remove the name from " +
	          "this route and use the name of the default child route for named " +
	          "links instead."
	        );
	      }
	    }
	    route.children.forEach(function (child) {
	      var childMatchAs = matchAs
	        ? cleanPath((matchAs + "/" + (child.path)))
	        : undefined;
	        //递归调用
	      addRouteRecord(pathList, pathMap, nameMap, child, record, childMatchAs);
	    });
	  }

如果route设置了别名，还会以别名作为路由的path来更新pathList、pathMap、nameMap：

	  //设置了别名
	  if (route.alias !== undefined) {
	    var aliases = Array.isArray(route.alias)
	      ? route.alias
	      : [route.alias];
	
	    aliases.forEach(function (alias) {
	      var aliasRoute = {
	        path: alias,
	        children: route.children
	      };
	      //递归调用
	      addRouteRecord(
	        pathList,
	        pathMap,
	        nameMap,
	        aliasRoute,
	        parent,
	        record.path || '/' // matchAs
	      );
	    });
	  }

更新pathList、pathMap的代码如下：

	  //是新的，跟新pathList、pathMap
	  //pathList里面存的是路径
	  //pathMap里面存的是record,路径为key
	  //
	  if (!pathMap[record.path]) {
	    pathList.push(record.path);
	    pathMap[record.path] = record;
	  }

如果route定义了name属性，则还会更新nameMap:

	if (name) {
	    //nameMap里面存的是record,路径为key
	    if (!nameMap[name]) {
	      nameMap[name] = record;
	    } else if ("development" !== 'production' && !matchAs) {
	      warn(
	        false,
	        "Duplicate named routes definition: " +
	        "{ name: \"" + name + "\", path: \"" + (record.path) + "\" }"
	      );
	    }
	  }

我们可以举个例子看看createRouteMap(routes)运行后得到的pathList、pathMap、nameMap：

	// 1. 定义（路由）组件。
	// 可以从其他文件 import 进来
	const Foo = {
	  template: '<div>foo</div>',
	  beforeRouteEnter (to, from, next) {
	    // 在渲染该组件的对应路由被 confirm 前调用
	    // 不！能！获取组件实例 `this`
	    // 因为当守卫执行前，组件实例还没被创建
	    console.log(to)
	    next()
	  },
	  beforeRouteUpdate (to, from, next) {
	    // 在当前路由改变，但是该组件被复用时调用
	    // 举例来说，对于一个带有动态参数的路径 /foo/:id，在 /foo/1 和 /foo/2 之间跳转的时候，
	    // 由于会渲染同样的 Foo 组件，因此组件实例会被复用。而这个钩子就会在这个情况下被调用。
	    // 可以访问组件实例 `this`
	  },
	  beforeRouteLeave (to, from, next) {
	    // 导航离开该组件的对应路由时调用
	    // 可以访问组件实例 `this`
	    console.log(to)
	    next()
	  }
	}
	const Bar = { template: '<div>bar</div>' }
	const BarChild = { template: '<div>barchild</div>' }
	// 2. 定义路由
	// 每个路由应该映射一个组件。 其中"component" 可以是
	// 通过 Vue.extend() 创建的组件构造器，
	// 或者，只是一个组件配置对象。
	// 我们晚点再讨论嵌套路由。
	const routes = [
	  {
	      path: '/foo',
	      name: 'name-foo',
	      component: Foo,
	      alias: '/alias-foo'
	  },
	  {
	      path: '/bar',
	      component: Bar,
	      children: [{
	           path: 'posts',
	           name: 'name-barchild',
	           component: BarChild
	      }]
	  }
	]
	
	// 3. 创建 router 实例，然后传 `routes` 配置
	// 你还可以传别的配置参数, 不过先这么简单着吧。
	const router = new VueRouter({
	  routes // （缩写）相当于 routes: routes
	})

当执行到createMatcher函数中的`var ref = createRouteMap(routes);`时，我们可以看看ref的结果：

![](/assets/vue-router2.png)

![](https://i.imgur.com/DGnJpRE.png)

每个path或name记录的都是一个record对象：

![](/assets/vue-router4.png)

![](https://i.imgur.com/CyW4j5p.png)

![](/assets/vue-router3.png)

![](https://i.imgur.com/Yb8Ua2M.png)

这里面还用到了normalizePath函数，该函数定义在vue-router源码的1264 ~ 1269行：

	/*
	 * 规范化路径
	 */
	function normalizePath (path, parent, strict) {
	  if (!strict) { path = path.replace(/\/$/, ''); }
	  if (path[0] === '/') { return path }
	  if (parent == null) { return path }
	  return cleanPath(((parent.path) + "/" + path))
	}

它所做的主要是对path做一定的检查：

首先当path以斜线/结尾的时候将斜线去除。然后判断path是否以/开头，如果是则直接返回。最后调用cleanPath对path做进一步的处理。

cleanPath定义在vue-router源码的645 ~ 647行，其参数是一个path，对于子路由的path而言，这里会用父亲的path和孩子的path以“/”连接起来作为新的path传进去处理：

	//净化path，主要就是将连续两个//换成一个
	//'a//b///c/d//f'.replace(/\/\//g, '/') "a/b//c/d/f"
	//并没法完全净化，只会净化一次
	function cleanPath (path) {
	  return path.replace(/\/\//g, '/')
	}

从代码可以看出，cleanPath的主要目的就是将连着的两个//替换成单个/。


**compileRouteRegex**

除此之外，上述函数还调用了compileRouteRegex函数，compileRouteRegex函数用于将path编译成正则表达式，它实际上调用的是定义在vue-router源代码的1059~1076行的pathToRegexp函数。又根据path的具体类型（正则、String、Array）分别调用了regexpToRegexp、arrayToRegexp、stringToRegexp函数来实现。具体据不再展开了，反正最终就是在record上定义了一个regex属性，该属性的值就是path对应的正则表达式，且该正则表达式有一个keys数组属性。而keys里面又存储着该正则表达式的一些tokens。我们还是以上面的例子的一个路由的结果来看看：

![](/assets/vue-router5.png)

![](https://i.imgur.com/ZrcT6K2.png)

#### 8.1.2 原型属性
除此成员属性之外，VueRouter还定了了prototypeAccessors原型属性：

	//定义原型属性
	Object.defineProperties( VueRouter.prototype, prototypeAccessors );

prototypeAccessors在vue-router源码的2465行定义如下：

	var prototypeAccessors = { currentRoute: { configurable: true } };

它上面还定义了一个get方法，用于拿到当前实例的history属性, 它的实现在vue-router源码的2475 ~ 2477行：

	prototypeAccessors.currentRoute.get = function () {
	  return this.history && this.history.current
	};


#### 8.1.3 原型方法
VueRouter类定义了`match`、`init`、`beforeEach`、`beforeResolve`、`afterEach`、`onReady`、`onError`、`push`、`replace`、`go`、`back`、`forward`、`getMatchedComponents`、`resolve`、`addRoutes` 15个原型方法。我们分别来看一下它的实现。

##### 8.1.3.1 VueRouter.prototype.match方法
VueRouter.prototype.match方法定已在源码的2467 ~ 2473行，用于匹配出目标路由，它实际上调用的是matcher实例的match方法，该方法我们在前面已经介绍过了。

	VueRouter.prototype.match = function match (
	  raw,
	  current,
	  redirectedFrom
	) {
	  return this.matcher.match(raw, current, redirectedFrom)
	};

##### 8.1.3.2 VueRouter.prototype.init方法
VueRouter.prototype.init方法定义在vue-router源码的2479 ~ 2517行，它实际上是在vue-router组件的install方法中被调用。
它一方面将vue实例赋给app属性，另一方面也将app放入apps数组。除此之外还会注册一些监听器：

	VueRouter.prototype.init = function init (app /* Vue component instance */) {
	    var this$1 = this;
	
	  "development" !== 'production' && assert(
	    install.installed,
	    "not installed. Make sure to call `Vue.use(VueRouter)` " +
	    "before creating root instance."
	  );
	
	  this.apps.push(app);
	
	  // main app already initialized.
	  if (this.app) {
	    return
	  }
	
	  this.app = app;
	
	  var history = this.history;
	
	  if (history instanceof HTML5History) {
	    history.transitionTo(history.getCurrentLocation());
	  } else if (history instanceof HashHistory) {
	    var setupHashListener = function () {
	      history.setupListeners();
	    };
	    history.transitionTo(
	      history.getCurrentLocation(),
	      setupHashListener,
	      setupHashListener
	    );
	  }
	
	  history.listen(function (route) {
	    this$1.apps.forEach(function (app) {
	      app._route = route;
	    });
	  });
	};

##### 8.1.3.3 VueRouter.prototype.beforeEach方法
可以用来注册一个全局前置守卫，我们看看官方文档对它的详细描述：

![](/assets/vue-router6.png)

![](https://i.imgur.com/eObRcwr.png)

该方法定义在vue-router源码的2519 ~ 2521行：

	//钩子函数
	//把fn加入到beforeHooks列表
	VueRouter.prototype.beforeEach = function beforeEach (fn) {
	  return registerHook(this.beforeHooks, fn)
	};

它实际上调用的是registerHook函数，用于在router实例的全局前置守卫属性beforeHooks上注册钩子函数fn。registerHook函数定义在vue-router源码的2609 ~ 2615行：

	function registerHook (list, fn) {
	  list.push(fn);//把fn加入到钩子函数
	  return function () {//返回一个函数，当函数被执行时用于从列表中移除fn
	    var i = list.indexOf(fn);
	    if (i > -1) { list.splice(i, 1); }
	  }
	}

我们可以看到，registerHook其实就是把钩子函数fn放到router实例的全局前置守卫属性beforeHooks数组中，并且返回一个函数，当这个返回的函数被执行时会将注册的fn从beforeHooks数组中移除，也就是实现解绑功能。
##### 8.1.3.4 VueRouter.prototype.beforeResolve方法
beforeResolve和beforeEach方法基本类似，可以用来注册一个全局解析守卫，我们看看官方文档对它的详细描述：

![](/assets/vue-router7.png)

![](https://i.imgur.com/giWB91H.png)

该方法定义在vue-router源码的2523 ~ 2525行：

	//把fn加入到resolveHooks列表
	VueRouter.prototype.beforeResolve = function beforeResolve (fn) {
	  return registerHook(this.resolveHooks, fn)
	};

它实际上调用的也是registerHook函数，用于在router实例的全局解析守卫属性resolveHooks上注册钩子函数fn。 registerHook函数在前面已经讲过了，这里不再赘述。

##### 8.1.3.5 VueRouter.prototype.afterEach方法
顾名思义，可以用来注册一个全局后置守卫，我们看看官方文档对它的详细描述：

![](/assets/vue-router8.png)

![](https://i.imgur.com/LrspUDC.png)

该方法定义在vue-router源码的2527 ~ 2529行：

	//把fn加入到afterHooks列表
	VueRouter.prototype.afterEach = function afterEach (fn) {
	  return registerHook(this.afterHooks, fn)
	};

它实际上调用的还是registerHook函数，用于在router实例的全局后置守卫属性afterHooks上注册钩子函数fn。 registerHook函数在前面已经讲过了，这里不再赘述。

##### 8.1.3.6 VueRouter.prototype.onReady方法
该方法定义在vue-router源码的xxx行，它接收cb, errorCb两个参数，用于注册ready时的回调。当History实例的当前ready状态为true时，会执行cb回调，否则将cb回调函数放入实例的readyCbs队列。如果提供了errorCb参数，则将errorCb加入readyErrorCbs队列。其源码如下：

	VueRouter.prototype.onReady = function onReady (cb, errorCb) {
	  this.history.onReady(cb, errorCb);
	};

可以看到，它实际上就是调用的History类中定义的onReady函数，这个函数我们在History类中已经详细介绍过。

##### 8.1.3.7 VueRouter.prototype.onError方法
VueRouter.prototype.onError方法定义在vue-router源码的2535 ~ 2537行，它接收一个errorCb参数，用于注册错误的回调。它会把errorCb加入到history实例的errorCbs队列。其源码如下：

	VueRouter.prototype.onError = function onError (errorCb) {
	  this.history.onError(errorCb);
	};

##### 8.1.3.8 VueRouter.prototype.push方法
VueRouter.prototype.push方法定义在源码的2539 ~ 2541行，它实际上调用的是history实例的push方法。相当于对三种History模式的push方法又做了一层封装：

	VueRouter.prototype.push = function push (location, onComplete, onAbort) {
	  this.history.push(location, onComplete, onAbort);
	};

##### 8.1.3.9 VueRouter.prototype.replace方法
VueRouter.prototype.replace方法定义在源码的2543 ~ 2545行，它实际上调用的是history实例的replace方法。相当于对三种History模式的replace方法又做了一层封装：

	VueRouter.prototype.replace = function replace (location, onComplete, onAbort) {
	  this.history.replace(location, onComplete, onAbort);
	};

##### 8.1.3.10 VueRouter.prototype.go方法
VueRouter.prototype.go方法定义在vue-router源码的2547 ~ 2549行，用于实现编程式地前进/后退，它实际上调用的是挂载在router实例上的history属性的go方法，而router实例上的history属性的go方法实际上调用的是window.history.go(n)，这一点我们在三种History类中都已经介绍过了。

	VueRouter.prototype.go = function go (n) {
	  this.history.go(n);
	};

##### 8.1.3.11 VueRouter.prototype.back方法
VueRouter.prototype.back定义在vue-router源码的2551 ~ 2553行，它实际上调用的是VueRouter.prototype.go(-1)，用于实现编程式地后退:

	VueRouter.prototype.back = function back () {
	  this.go(-1);
	};


##### 8.1.3.12 VueRouter.prototype.forward方法
VueRouter.prototype.forward方法定义在vue-router源码的2555 ~ 2557行，它实际上调用的是VueRouter.prototype.go(1)，用于实现编程式地前进:

	VueRouter.prototype.forward = function forward () {
	  this.go(1);
	};


##### 8.1.3.13 VueRouter.prototype.getMatchedComponents方法
VueRouter.prototype.getMatchedComponents方法定义在源码的2559 ~ 2573行，它返回目标位置或是当前路由匹配的组件数组（是数组的定义/构造类，不是实例）。通常在服务端渲染的数据预加载时时候。

	VueRouter.prototype.getMatchedComponents = function getMatchedComponents (to) {
	  var route = to
	    ? to.matched
	      ? to
	      : this.resolve(to).route
	    : this.currentRoute;
	  if (!route) {
	    return []
	  }
	  return [].concat.apply([], route.matched.map(function (m) {
	    return Object.keys(m.components).map(function (key) {
	      return m.components[key]
	    })
	  }))
	};

我们来看看前面使用过的例子：

	let router
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
        console.log(router.getMatchedComponents())
        // if the returned position is falsy or an empty object,
        // will retain current scroll position.
        return position
      }
    }

    router = new VueRouter({
      mode: 'hash',
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
            <li><router-link tag="div" to="/bar#anchor"><a>/bar#anchor</a></router-link></li>
          </ul>
          <router-view class="view"></router-view>
        </div>
      `
    }).$mount('#app')

点击各个路由，得到的输出结果如下：

![](/assets/vue-router9.png)

![](https://i.imgur.com/0VcqbZG.png)

##### 8.1.3.14 VueRouter.prototype.resolve方法
resolve方法用于解析目标位置，它定义在vue-router源码的2575 ~ 2598行，它通常用在路由跳转的时候，这个我们在“**router-link组件**”这一章节介绍过，它主要是根据目标location，当前路由来解析出一个关于目标路由的对象，这个对象包含目标路由、location以及href:

	VueRouter.prototype.resolve = function resolve (
	  to,
	  current,
	  append
	) {
	  var location = normalizeLocation(
	    to,
	    current || this.history.current,
	    append,
	    this
	  );
	  var route = this.match(location, current);
	  var fullPath = route.redirectedFrom || route.fullPath;
	  var base = this.history.base;
	  var href = createHref(base, fullPath, this.mode);
	  return {
	    location: location,
	    route: route,
	    href: href,
	    // for backwards compat
	    normalizedTo: location,
	    resolved: route
	  }
	};

##### 8.1.3.15 VueRouter.prototype.addRoutes方法
VueRouter.prototype.addRoutes方法用于动态添加更多的路由规则。参数必须是一个符合 routes 选项要求的数组。我们还记得，router实例的matcher是一个对象，这个对象有一个addRoutes方法，VueRouter.prototype.addRoutes方法就是调用的前述的方法完成的。addRoutes方法的最终还是调用的createRouteMap生成路由映射。因为我们在初始化matcher时就已经创建过一次路由映射了，所以这里相当于就是更新路由映射，即更新pathLsit、pathMap、nameMap三个数组。

	VueRouter.prototype.addRoutes = function addRoutes (routes) {
	  this.matcher.addRoutes(routes);
	  if (this.history.current !== START) {
	    this.history.transitionTo(this.history.getCurrentLocation());
	  }
	};
