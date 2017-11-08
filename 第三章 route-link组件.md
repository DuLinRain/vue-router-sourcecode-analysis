# 第三章 route-link组件
router-link组件定义在vue-router源码的385 ~ 488行，我们来看看它的实现：

	// work around weird flow bug
	var toTypes = [String, Object];
	var eventTypes = [String, Array];
	//Link组件,对应的是router-link,貌似还是函数组件
	var Link = {
	  name: 'router-link',
	  props: {
	    //表示目标路由的链接。当被点击后，内部会立刻把 to 的值传到 router.push()，所以这个值可以是一个字符串或者是描述目标位置的对象。
	    to: {
	      type: toTypes,
	      required: true
	    },
	    //有时候想要 <router-link> 渲染成某种标签，例如 <li>。 于是我们使用 tag prop 类指定何种标签，同样它还是会监听点击，触发导航。
	    tag: {
	      type: String,
	      default: 'a'
	    },
	    //"是否激活" 默认类名的依据是 inclusive match （全包含匹配）。 举个例子，如果当前的路径
		//是 /a 开头的，那么 <router-link to="/a"> 也会被设置 CSS 类名。
	    //"是否激活" 默认类名的依据是 inclusive match （全包含匹配）。 举个例子，如果当前的路径
		//是 /a 开头的，那么 <router-link to="/a"> 也会被设置 CSS 类名。
	    exact: Boolean,
	    //设置 append 属性后，则在当前（相对）路径前添加基路径。例如，我们从 /a 导航到一个
		//相对路径 b，如果没有配置 append，则路径为 /b，如果配了，则为 /a/b
	    append: Boolean,
	    //设置 replace 属性的话，当点击时，会调用 router.replace() 而不是 router.push()，
		//于是导航后不会留下 history 记录。
	    replace: Boolean,
	    //设置 链接激活时使用的 CSS 类名。默认值可以通过路由的构造选项 linkActiveClass 来全局配置
	    activeClass: String,
	    //配置当链接被精确匹配的时候应该激活的 class。注意默认值也是可以通过路由构造函数
		//选项 linkExactActiveClass 进行全局配置的。
	    exactActiveClass: String,
	    //声明可以用来触发导航的事件。可以是一个字符串或是一个包含字符串的数组。
	    event: {
	      type: eventTypes,
	      default: 'click'
	    }
	  },
	  render: function render (h) {
	    var this$1 = this;
	
	    var router = this.$router;//路由实例？
	    var current = this.$route;//当前的路由
	    //解析目标位置（格式和 <router-link> 的 to prop 一样），返回包含如下属性的对象：
	    // {
	    //   location: Location;
	    //   route: Route;
	    //   href: string;
	    // }
	    // current 是当前默认的路由 (通常你不需要改变它)
	    // append 允许你在 current 路由上附加路径 (如同 router-link)
	    var ref = router.resolve(this.to, current, this.append);
	    var location = ref.location;
	    var route = ref.route;
	    var href = ref.href;
	
	    var classes = {};
	    //这是在哪里配置的啊？？
	    var globalActiveClass = router.options.linkActiveClass;//匹配到后的全局类
	    var globalExactActiveClass = router.options.linkExactActiveClass;//精准匹配到时的全局类
	    // Support global empty active class
	    var activeClassFallback = globalActiveClass == null
	            ? 'router-link-active'
	            : globalActiveClass;
	    var exactActiveClassFallback = globalExactActiveClass == null
	            ? 'router-link-exact-active'
	            : globalExactActiveClass;
	    var activeClass = this.activeClass == null
	            ? activeClassFallback
	            : this.activeClass;
	    var exactActiveClass = this.exactActiveClass == null
	            ? exactActiveClassFallback
	            : this.exactActiveClass;
	    var compareTarget = location.path
	      ? createRoute(null, location, null, router)
	      : route;
	
	    classes[exactActiveClass] = isSameRoute(current, compareTarget);
	    //activeClass的true或false与exact、isIncludedRoute(current, compareTarget)有关
	    classes[activeClass] = this.exact
	      ? classes[exactActiveClass]
	      : isIncludedRoute(current, compareTarget);
	
	    var handler = function (e) {
	      if (guardEvent(e)) {//守护事件
	        if (this$1.replace) {//如果指定了replace
	          router.replace(location);
	        } else {
	          router.push(location);
	        }
	      }
	    };
	
	    var on = { click: guardEvent };//默认click事件是guardEvent
	    //为事件指定响应函数
	    if (Array.isArray(this.event)) {//如果指定了几种事件
	      this.event.forEach(function (e) { on[e] = handler; });
	    } else {
	      on[this.event] = handler;
	    }
	    //data用于渲染dom，creatElement
	    var data = {
	      class: classes//点击后的类
	    };
	
	    if (this.tag === 'a') {//指定了渲染标签类型为a标签
	      data.on = on;
	      data.attrs = { href: href };//渲染成a link，设置href
	    } else {//其它情况
	      // find the first <a> child and apply listener and href
	      // 看slot里面有没有a，如果有的话应用在第一个a上面
	      // 很好奇这个东西this.$slots.default是什么？？对象？？
	      var a = findAnchor(this.$slots.default);
	      if (a) {
	        // in case the <a> is a static node
	        // 避免a是一个静态节点
	        a.isStatic = false;
	        var extend = _Vue.util.extend;//用的是vue的extend方法
	        var aData = a.data = extend({}, a.data);//拿到a原有的数据
	        aData.on = on;//拿到事件
	        var aAttrs = a.data.attrs = extend({}, a.data.attrs);//拿到a原有的属性
	        aAttrs.href = href;//设置href
	      } else {
	        // doesn't have <a> child, apply listener to self
	        // 没有a的话，监听自己
	        data.on = on;
	      }
	    }
	    //渲染dom
	    return h(this.tag, data, this.$slots.default)
	  }
	};

从该组件的props可以看出，只有to属性是必须的。还可以看出router-link默认会渲染成a标签，默认的触发行为是click。具体的props官方文档描述的非常详细，我们可以看看：

![](/assets/2-4.png)

![](https://i.imgur.com/3rzqqDA.png)

![](/assets/2-5.png)

![](https://i.imgur.com/sMPQbKW.png)

![](/assets/2-6.png)

![](https://i.imgur.com/dLZ01r6.png)

![](/assets/2-7.png)

![](https://i.imgur.com/IN7rylV.png)

可以看到，虽然从props属性来看，只有tag和event有默认值，但是官方文档却说其它一些属性都有对应的默认值，这是为啥呢？其实这些默认值都是在render函数中指定的。 我们来看看render函数。

render函数首先拿到router实例以及当前页面的路由，然后根据当前route、目标route以及append属性来解析出：

	var router = this.$router;
	var current = this.$route;
	var ref = router.resolve(this.to, current, this.append);

resolve函数定义在源码的2575 ~ 2598行,它根据当前route、目标route以及append属性来解析出一个包含location、route、href属性的对象：

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

该方法首先会调用normalizeLocation函数进行规范化，normalizeLocation函数除了接收resolve函数的三个参数外，还会将router实例作为第四个参数传递进去，normalizeLocation定义在vuex源码的1274 ~ 1326行：

	function normalizeLocation (
	  raw,
	  current,
	  append,
	  router
	) {
	  var next = typeof raw === 'string' ? { path: raw } : raw;
	  // named target
	  if (next.name || next._normalized) {
	    return next
	  }
	
	  // relative params
	  if (!next.path && next.params && current) {
	    next = assign({}, next);
	    next._normalized = true;
	    var params = assign(assign({}, current.params), next.params);
	    if (current.name) {
	      next.name = current.name;
	      next.params = params;
	    } else if (current.matched.length) {
	      var rawPath = current.matched[current.matched.length - 1].path;
	      next.path = fillParams(rawPath, params, ("path " + (current.path)));
	    } else {
	      warn(false, "relative params navigation requires a current route.");
	    }
	    return next
	  }
	
	  var parsedPath = parsePath(next.path || '');
	  var basePath = (current && current.path) || '/';
	  var path = parsedPath.path
	    ? resolvePath(parsedPath.path, basePath, append || next.append)
	    : basePath;
	
	  var query = resolveQuery(
	    parsedPath.query,
	    next.query,
	    router && router.options.parseQuery
	  );
	
	  var hash = next.hash || parsedPath.hash;
	  if (hash && hash.charAt(0) !== '#') {
	    hash = "#" + hash;
	  }
	
	  return {
	    _normalized: true,
	    path: path,
	    query: query,
	    hash: hash
	  }
	}

normalizeLocation首先会判断<router=link>的to属性，我们知道官方文档曾说：
>to表示目标路由的链接。当被点击后，内部会立刻把 to 的值传到 router.push()，所以这个值可以是一个**字符串**或者是**描述目标位置的对象**。

normalizeLocation函数中，判断到to属性是一个字符串的时候会将它转换成一个对象，该字符串作为path属性的值。当to属性本身是一个对象时，会检测是否具有name属性或者是否已经规范化了(_normalized为true)。如果任何一种情况成立都会直接返回。否则则当to不存在path属性但包含params属性时，会合并current和to的params，然后会分以下几种情况：

1. 如果current有name属性，则将current.name赋给to.name，将params赋给to.params
2. 否则，如果current的matched属性不为空数组，则取最后一个元素的path，调用fillParams来填充to.path属性。

然后返回to。

而如果“**to不存在path属性但包含params属性时**”这个属性不成立，则会根据to.path解析出原始path:

	var parsedPath = parsePath(next.path || '');

parsePath函数定义在源码的622 ~ 643行：

	function parsePath (path) {
	  var hash = '';
	  var query = '';
	
	  var hashIndex = path.indexOf('#');
	  if (hashIndex >= 0) {
	    hash = path.slice(hashIndex);
	    path = path.slice(0, hashIndex);
	  }
	
	  var queryIndex = path.indexOf('?');
	  if (queryIndex >= 0) {
	    query = path.slice(queryIndex + 1);
	    path = path.slice(0, queryIndex);
	  }
	
	  return {
	    path: path,
	    query: query,
	    hash: hash
	  }
	}

它通过原始path解析出query、hash、path，这里的path是去除query和hash后的。

然后以current的path作为基准路径：

	var basePath = (current && current.path) || '/';

最后根据去除query和hash后的path（parsedPath）、基准path（basePath）和append属性来调用resolvePath解析出最终的路径，resolvePath定义在源码的580 ~ 620行：

	function resolvePath (
	  relative,
	  base,
	  append
	) {
	  var firstChar = relative.charAt(0);
	  if (firstChar === '/') {
	    return relative
	  }
	
	  if (firstChar === '?' || firstChar === '#') {
	    return base + relative
	  }
	
	  var stack = base.split('/');
	
	  // remove trailing segment if:
	  // - not appending
	  // - appending to trailing slash (last segment is empty)
	  if (!append || !stack[stack.length - 1]) {
	    stack.pop();
	  }
	
	  // resolve relative path
	  var segments = relative.replace(/^\//, '').split('/');
	  for (var i = 0; i < segments.length; i++) {
	    var segment = segments[i];
	    if (segment === '..') {
	      stack.pop();
	    } else if (segment !== '.') {
	      stack.push(segment);
	    }
	  }
	
	  // ensure leading slash
	  if (stack[0] !== '') {
	    stack.unshift('');
	  }
	
	  return stack.join('/')
	}

类似的会得到query和hash：

	var query = resolveQuery(
	    parsedPath.query,
	    next.query,
	    router && router.options.parseQuery
	);

    var hash = next.hash || parsedPath.hash;
    if (hash && hash.charAt(0) !== '#') {
    	hash = "#" + hash;
    }
这里面会调用一个resolveQuery函数，该函数用于将字符串形式的query解析成key-value形式，当有多个相同key的时候，value会是一个数组，该函数定义在vue-router源码的164 ~ 183行：

	//解析query
	//返回一个query，但是已经转成了key-value形式
	//实际上还是调用的下面的parseQuery函数
	//只不过它支持extraQuery,_parseQuery这两个额外的参数
	function resolveQuery (
	  query,
	  extraQuery,//额外的请求对象，是一个key-value的
	  _parseQuery//自定义解析函数
	) {
	  if ( extraQuery === void 0 ) extraQuery = {};
	
	  var parse = _parseQuery || parseQuery;
	  var parsedQuery;
	  try {
	    parsedQuery = parse(query || '');
	  } catch (e) {
	    "development" !== 'production' && warn(false, e.message);
	    parsedQuery = {};
	  }
	  for (var key in extraQuery) {
	    parsedQuery[key] = extraQuery[key];//覆盖/增加
	  }
	  return parsedQuery
	}

可以看到，它接收第三个参数用于指定解析方式，而如果没有指定，则会默认使用parseQuery方法，该方法定义在vue-router源码的185 ~ 211行：

	// 解析成一个对象，key-value形式
	// 真正的解析query的函数
	function parseQuery (query) {
	  var res = {};
	
	  query = query.trim().replace(/^(\?|#|&)/, '');//将这几个字符都替换成空字符
	
	  if (!query) {
	    return res
	  }
	
	  query.split('&').forEach(function (param) {
	    var parts = param.replace(/\+/g, ' ').split('=');//首先分隔=
	    var key = decode(parts.shift());//解码key
	    var val = parts.length > 0//解码value
	      ? decode(parts.join('='))
	      : null;
	    //将name=linyu&name=liming&age=20这样的query字符串转换成
	    //{name: ['linyu', 'liming'], age: 20}这样的query对象
	    if (res[key] === undefined) {
	      res[key] = val;
	    } else if (Array.isArray(res[key])) {
	      res[key].push(val);
	    } else {
	      res[key] = [res[key], val];
	    }
	  });
	
	  return res
	}

可以看到它才是真正实现将query解析成一个对象的函数。这里面还用到了一个decode方法，用于解析编码，采用的就是原生的decodeURIComponent，这个在162行有定义：

	var decode = decodeURIComponent;

query从字符串转对象的逆过程也有一个对应的方法，这个方法就是stringifyQuery,它定义在vue-router源码的213 ~ 243行，在后面的createRoute函数中会用到，在这里提前放出来：

	//是parseQuery的逆过程，将{name: ['linyu', 'liming'], age: 20}之类的query对象
	//转成?name=linyu&name=liming&age=20这样的query字符串
	function stringifyQuery (obj) {
	  var res = obj ? Object.keys(obj).map(function (key) {
	    var val = obj[key];
	
	    if (val === undefined) {
	      return ''
	    }
	
	    if (val === null) {
	      return encode(key)
	    }
	
	    if (Array.isArray(val)) {
	      var result = [];
	      val.forEach(function (val2) {
	        if (val2 === undefined) {
	          return
	        }
	        if (val2 === null) {
	          result.push(encode(key));
	        } else {
	          result.push(encode(key) + '=' + encode(val2));
	        }
	      });
	      return result.join('&')
	    }
	
	    return encode(key) + '=' + encode(val)
	  }).filter(function (x) { return x.length > 0; }).join('&') : null;
	  return res ? ("?" + res) : ''
	}

这里面还会调用encode方法，与decode方法不同的是，encode方法调用的并不是原生的encodeURIComponent, 而是对它做了进一步的处理，将encodeURIComponent没有转义的字符`!'()*`也进行了转义（151 ~ 160 行）：

	/*  */
	//匹配!'()*这几个符号
	var encodeReserveRE = /[!'()*]/g;
	var encodeReserveReplacer = function (c) { return '%' + c.charCodeAt(0).toString(16); };
	var commaRE = /%2C/g;//逗号
	
	// fixed encodeURIComponent which is more conformant to RFC3986:
	// - escapes [!'()*]
	// - preserve commas
	// 也就是说encodeURIComponent默认不会转义!'()*这几个符号，会转义,号
	// encode要做的是转义!'()*这几个符号，不转义,号
	// 自定义encode函数
	var encode = function (str) { return encodeURIComponent(str)
	  .replace(encodeReserveRE, encodeReserveReplacer)
	  .replace(commaRE, ','); };

回到normalizeLocation函数，最终normalizeLocation函数会返回一个包含\_normalized、path、query、hash的对象：

	return {
	    _normalized: true,
	    path: path,
	    query: query,
	    hash: hash
	}

回到resolve函数，在normalizeLocation函数调用后，接下来调用的是match方法，

	var route = this.match(location, current);

match方法用于根据location和当前路由得出location对应的路由，它实际上调用的是router实例上**matcher属性的match方法**(源码2467 ~ 2473行)：

	VueRouter.prototype.match = function match (
	  raw,
	  current,
	  redirectedFrom
	) {
	  return this.matcher.match(raw, current, redirectedFrom)
	};

**matcher属性的match方法**实际上是定义在createMatcher函数中(源码1338 ~ 1501行)定义的, 该函数是在router实例初始化的时候调用的，所做的工作主要是根据routes生成三个重要的数据结构：pathList(path组成的数组)、pathMap(path->record的映射)、nameMap(name->record的映射)。最终就是返回一个匹配的路由。

回到resolve函数，它会继续拿到redirectedFrom，base，mode等属性生成一个对应模式下的超链接：

	var fullPath = route.redirectedFrom || route.fullPath;
	var base = this.history.base;
	var href = createHref(base, fullPath, this.mode);

最终resolve函数会返回一个对象，包含三个主要属性：规范化后的location、匹配到的route、超链接href:

	return {
	    location: location,
	    route: route,
	    href: href,
	    // for backwards compat
	    normalizedTo: location,
	    resolved: route
	}

回到router-link组件，在拿到to配到到的路由之后，它会定义classes对象：

	var classes = {};
    var globalActiveClass = router.options.linkActiveClass;
    var globalExactActiveClass = router.options.linkExactActiveClass;
    // Support global empty active class
    var activeClassFallback = globalActiveClass == null
            ? 'router-link-active'
            : globalActiveClass;
    var exactActiveClassFallback = globalExactActiveClass == null
            ? 'router-link-exact-active'
            : globalExactActiveClass;
    var activeClass = this.activeClass == null
            ? activeClassFallback
            : this.activeClass;
    var exactActiveClass = this.exactActiveClass == null
            ? exactActiveClassFallback
            : this.exactActiveClass;
    var compareTarget = location.path
      ? createRoute(null, location, null, router)
      : route;

    classes[exactActiveClass] = isSameRoute(current, compareTarget);
    classes[activeClass] = this.exact
      ? classes[exactActiveClass]
      : isIncludedRoute(current, compareTarget);

其实主要定义了classes的两个属性：activeClass、exactActiveClass。这两个属性可以在使用router-link组件时通过props指定，如果没有指定也可以使用VueRouter实例化时的options配置项指定linkActiveClass、linkExactActiveClass，这两个属性官方文档都有描述：

![](/assets/2-8.png)

![](https://i.imgur.com/nD1gxpd.png)

而当用户既没有通过router-link组件的props属性也没有通过options指定时，会分别有默认值：router-link-active、router-link-exact-active。

回到router-link组件，它在location.path存在的时候回调用createRoute函数创建一个route，这个函数定义在源码的250 ~ 277行：

	//创建路由
	function createRoute (
	  record,//
	  location,//
	  redirectedFrom,//
	  router//路由实例
	) {
	  var stringifyQuery$$1 = router && router.options.stringifyQuery;//拿到路由实例上挂载的stringifyQuery
	
	  var query = location.query || {};
	  try {
	    query = clone(query);//克隆一个？？
	  } catch (e) {}
	  //创建的route包含name、meta、path、hash、query、params、fullPath、matched等属性
	  var route = {//路由信息对象，一条route??
	    name: location.name || (record && record.name),
	    meta: (record && record.meta) || {},
	    path: location.path || '/',
	    hash: location.hash || '',
	    query: query,
	    params: location.params || {},
	    fullPath: getFullPath(location, stringifyQuery$$1),
	    matched: record ? formatMatch(record) : []
	  };
	  //如果有redirectedFrom参数，还会设置route的该属性
	  if (redirectedFrom) {
	    route.redirectedFrom = getFullPath(redirectedFrom, stringifyQuery$$1);
	  }
	  return Object.freeze(route)//冻结这个对象
	}

主要根据location得到一个**路由信息对象**（详细可见[官方文档](https://router.vuejs.org/zh-cn/api/route-object.html)），这里面会调用前面介绍过的stringifyQuery函数对query进行序列化操作。除此之外还调用了clone函数进行克隆操作，该函数定义在源码的279 ~ 291行：

	//经典的克隆方法，不过这里针对数组使用map调用自己挺讨巧的
	function clone (value) {
	  if (Array.isArray(value)) {
	    return value.map(clone)
	  } else if (value && typeof value === 'object') {
	    var res = {};
	    for (var key in value) {
	      res[key] = clone(value[key]);
	    }
	    return res
	  } else {
	    return value
	  }
	}

随后在classes对象上定义exactActiveClass属性时会调用isSameRoute函数，该函数定义在vue-router的319 ~ 340行，它主要是根据两个路由信息对象的属性判断他们是否是同一个路由：

	//判断两个route是不是同一个
	function isSameRoute (a, b) {
	  if (b === START) {//如果b就是传的START
	    return a === b
	  } else if (!b) {//如果没有传递b
	    return false
	  } else if (a.path && b.path) {//path都有
	    //path去掉反斜线后一样，hash一样，query一样
	    //
	    return (
	      a.path.replace(trailingSlashRE, '') === b.path.replace(trailingSlashRE, '') &&
	      a.hash === b.hash &&
	      isObjectEqual(a.query, b.query)//对象比较
	    )
	  } else if (a.name && b.name) {//path不存在 && 名字都存在
	    //名字一样、hash一样、query一样、params一样
	    return (
	      a.name === b.name &&
	      a.hash === b.hash &&
	      isObjectEqual(a.query, b.query) &&
	      isObjectEqual(a.params, b.params)
	    )
	  } else {
	    return false
	  }
	}

这里面还调用了isObjectEqual用于判断两个对象是否相等，我们知道对象没法像常规的数据那样去直接比较，所以这里是递归的判断对象的key-value是否相等，这个函数在很多场合（包括其他库）中都出现过：

	//判断对象的key-value是否一样，并不是普通意义上的直接===比较
	//因为两个对象是永远不可能一样的
	function isObjectEqual (a, b) {
	  if ( a === void 0 ) a = {};
	  if ( b === void 0 ) b = {};
	
	  // handle null value #1566
	  if (!a || !b) { return a === b }
	  var aKeys = Object.keys(a);
	  var bKeys = Object.keys(b);
	  if (aKeys.length !== bKeys.length) {
	    return false
	  }
	  return aKeys.every(function (key) {
	    var aVal = a[key];
	    var bVal = b[key];
	    // check nested equality
	    if (typeof aVal === 'object' && typeof bVal === 'object') {
	      return isObjectEqual(aVal, bVal)
	    }
	    return String(aVal) === String(bVal)
	  })
	}

而在classes对象上定义activeClass属性时会调用isIncludedRoute函数，该函数定义在vue-router的364 ~ 372行，它主要是根据两个路由信息对象的属性判断他们是否存在包含关系：

	//路由是否是包含关系，比如：
	//'baidu.com/vuex/age/'.replace(trailingSlashRE, '/')
	//.indexOf('baidu.com/vuex/'.replace(trailingSlashRE, '/'))
	//&& target无hash或target的hash和current的hash相同
	//&& current的query包含target的query
	function isIncludedRoute (current, target) {
	  return (
	    current.path.replace(trailingSlashRE, '/').indexOf(
	      target.path.replace(trailingSlashRE, '/')
	    ) === 0 &&
	    (!target.hash || current.hash === target.hash) &&//target无hash或target的hash和current的hash相同
	    queryIncludes(current.query, target.query)//current的query包含target的query
	  )
	}
	//target有的key，current都有，则认为是included
	function queryIncludes (current, target) {
	  for (var key in target) {
	    if (!(key in current)) {
	      return false
	    }
	  }
	  return true
	}

回到router-link组件，它会注册响应的事件响应函数：

	var handler = function (e) {
      if (guardEvent(e)) {
        if (this$1.replace) {
          router.replace(location);
        } else {
          router.push(location);
        }
      }
    };

    var on = { click: guardEvent };
    if (Array.isArray(this.event)) {
      this.event.forEach(function (e) { on[e] = handler; });
    } else {
      on[this.event] = handler;
    }

因为props属性支持数组，也就是说用户可以指定多种触发事件，所以这里需要遍历event数组，on是一个对象，用于渲染dom时绑定事件。on初始化click事件为guardEvent，当然，在接下来遍历event数组之后event中的所有事件响应函数都会是handler函数了。实际上hander函数必定会调用guardEvent，guardEvent定义在源码的490 ~ 507行，主要对事件进行守卫：

	function guardEvent (e) {
	  // don't redirect with control keys
	  if (e.metaKey || e.altKey || e.ctrlKey || e.shiftKey) { return }
	  // don't redirect when preventDefault called
	  if (e.defaultPrevented) { return }
	  // don't redirect on right click
	  if (e.button !== undefined && e.button !== 0) { return }
	  // don't redirect if `target="_blank"`
	  if (e.currentTarget && e.currentTarget.getAttribute) {
	    var target = e.currentTarget.getAttribute('target');
	    if (/\b_blank\b/i.test(target)) { return }
	  }
	  // this may be a Weex event which doesn't have this method
	  if (e.preventDefault) {
	    e.preventDefault();
	  }
	  return true
	}

guardEvent返回true时才会执行相应的路由跳转操作，路由跳转会根据router-link组件是否指定replace属性来决定调用router的push方法还是replace方法。

接下来会完成dom的渲染操作的准备：

	var data = {
      class: classes
    };

    if (this.tag === 'a') {
      data.on = on;
      data.attrs = { href: href };
    } else {
      // find the first <a> child and apply listener and href
      var a = findAnchor(this.$slots.default);
      if (a) {
        // in case the <a> is a static node
        a.isStatic = false;
        var extend = _Vue.util.extend;
        var aData = a.data = extend({}, a.data);
        aData.on = on;
        var aAttrs = a.data.attrs = extend({}, a.data.attrs);
        aAttrs.href = href;
      } else {
        // doesn't have <a> child, apply listener to self
        data.on = on;
      }
    }

    return h(this.tag, data, this.$slots.default)

我们知道，router-link组件通常被渲染成一个a标签，但是并不是说必须是a标签，也可以是其它标签，所以这里更具不同的标签确定了不同的数据绑定方式。最终调用的就是vue的createElement函数进行的。这里面当用户指定的不是a标签的时候，它还是会尝试地在slot中去找是否有a标签，如果有的话会绑定到slot中的第一个a标签，没有则绑定到自身。这里面找slot中的a标签是递归调用findAnchor完成的, 它定义在源码的509 ~ 522行：

	function findAnchor (children) {
	  if (children) {
	    var child;
	    for (var i = 0; i < children.length; i++) {
	      child = children[i];
	      if (child.tag === 'a') {
	        return child
	      }
	      if (child.children && (child = findAnchor(child.children))) {
	        return child
	      }
	    }
	  }
	}

**举个例子：**

**没有指定tag时(同指定tag为a等同)**：

	<li><router-link to="/bar#anchor">/bar#anchor</router-link></li>

![](/assets/2-9.png)

![](https://i.imgur.com/qDMjCFs.png)

**有指定tag, 但不为a，且slot中无a标签**：

	<li><router-link tag="div" to="/bar#anchor">/bar#anchor</router-link></li>

![](/assets/2-10.png)

![](https://i.imgur.com/ynCRVvW.png)

**有指定tag, 但不为a，且slot中有a标签**：

	<li><router-link tag="div" to="/bar#anchor"><a>/bar#anchor</a></router-link></li>

![](/assets/2-11.png)

![](https://i.imgur.com/C9CZ7iv.png)



