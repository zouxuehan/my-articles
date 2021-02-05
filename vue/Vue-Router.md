### Vue-Router

vue使用插件的时候首先需要调用`Vue.use(plugin)`：

1. 首先维护一个数组`installedPlugins`存储所有的插件，
2. 调用插件的`install`方法，并将Vue作为参数传入，这样插件中就不用额外`import Vue`了
3. 最后将plugin存入数组

然后执行vue-router的`install`方法：

1. 调用`Vue.mixin`，在beforeCreate钩子函数中将实例化router存储在`Vm._router`中并执行初始化操作`init`然后`registerInstance`，在`destroyed`钩子函数中也`registerInstance`

   ```js
   Vue.mixin({
       beforeCreate () {
         if (isDef(this.$options.router)) {
           this._routerRoot = this
           this._router = this.$options.router
           this._router.init(this)
           Vue.util.defineReactive(this, '_route', this._router.history.current)
         } else {
           this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
         }
         registerInstance(this, this)
       },
       destroyed () {
         registerInstance(this)
       }
     })
   ```

   Vue.mixin会将传入的options合并到`Vue.options`中，然后每个组件创建构造函数的时候又会将`Vue.options`合并到自身`options`中，相当于所有的组件的这两个钩子中都有这些方法。

2. 然后在Vue原型中定义了`$router`和`$route`两个属性

   ```js
   Object.defineProperty(Vue.prototype, '$router', {
       get () { return this._routerRoot._router }
   })
   
   Object.defineProperty(Vue.prototype, '$route', {
       get () { return this._routerRoot._route }
   })
   ```

3. 全局注册<router-link>和<router-view>两个组件

   ```
   Vue.component('RouterView', View)
   Vue.component('RouterLink', Link)
   ```

#### new VueRouter

在vue中使用router时，会将router实例传入vue.options中，

```js
const routes = [
  { path: '/foo', component: Foo },
  { path: '/bar', component: Bar }
]
const router = new VueRouter({
  routes // (缩写) 相当于 routes: routes
})
const app = new Vue({
  router
}).$mount('#app')
```

在实例化VueRouter传入Vue实例之后，只有Vue实例执行`beforeCreate` 钩子的时候，会执行`this._router.init(this)`，将vue实例传入

```js
init (app: any /* Vue component instance */) {
   
    this.apps.push(app)

    // set up app destroyed handler
    // https://github.com/vuejs/vue-router/issues/2639
    app.$once('hook:destroyed', () => {
        // clean out app from this.apps array once destroyed
        const index = this.apps.indexOf(app)
        if (index > -1) this.apps.splice(index, 1)
        // ensure we still have a main app or null if no apps
        // we do not release the router so it can be reused
        if (this.app === app) this.app = this.apps[0] || null

        if (!this.app) this.history.teardown()
    })

    // main app previously initialized
    // return as we don't need to set up new history listener
    if (this.app) {
        return
    }

    this.app = app

    const history = this.history

    if (history instanceof HTML5History || history instanceof HashHistory) {
        ...
    }

    history.listen(route => {
        this.apps.forEach(app => {
            app._route = route
        })
    })
}
```

只有根Vue实例执行init，所有this.app存储根Vue实例，`this.apps` 保存持有 `$options.router` 属性的 `Vue`组件实例

然后根据不同类型的history执行不同的逻辑，最常使用的是`hash`路由模式，此时`this.history=new HashHistory(this, options.base, this.fallback)`执行以下逻辑

```js
const handleInitialScroll = routeOrError => {
    const from = history.current
    const expectScroll = this.options.scrollBehavior
    const supportsScroll = supportsPushState && expectScroll

    if (supportsScroll && 'fullPath' in routeOrError) {
        handleScroll(this, routeOrError, from, false)
    }
}
const setupListeners = routeOrError => {
    history.setupListeners()
    handleInitialScroll(routeOrError)
}
history.transitionTo(
    history.getCurrentLocation(),
    setupListeners,
    setupListeners
)
```

主要执行`history.transitionTo`方法，该处传入`location=history.getCurrentLocation()`，实际调用了`getHash`方法返回`window.location.href`url地址中 '#'后的字符串

```js
transitionTo (
    location: RawLocation,
    onComplete?: Function,
    onAbort?: Function
  ) {
    let route
    // catch redirect option https://github.com/vuejs/vue-router/issues/3201
    try {
      route = this.router.match(location, this.current)
    } catch (e) {
        ...
    }
    const prev = this.current
    this.confirmTransition(
     ...
    )
}
```

#### match匹配

首先执行`route = this.router.match(location, this.current)`将匹配的值赋值给route，实际调用`this.matcher.match`

```js
this.matcher = createMatcher(options.routes || [], this)

match (raw: RawLocation, current?: Route, redirectedFrom?: Location): Route {
    return this.matcher.match(raw, current, redirectedFrom)
}
```

来看`createMatcher`方法

```js
export function createMatcher (
  routes: Array<RouteConfig>,
  router: VueRouter
): Matcher {
  const { pathList, pathMap, nameMap } = createRouteMap(routes)
  
  function addRoutes (routes) {
    createRouteMap(routes, pathList, pathMap, nameMap)
  }
  function match(){ }
  ...
  return {match,addRoutes}
}
```

* 首先调用`createRouteMap(routes)`

  ```js
  export function createRouteMap (
    routes: Array<RouteConfig>,
    oldPathList?: Array<string>,
    oldPathMap?: Dictionary<RouteRecord>,
    oldNameMap?: Dictionary<RouteRecord>
  ): {
    pathList: Array<string>,
    pathMap: Dictionary<RouteRecord>,
    nameMap: Dictionary<RouteRecord>
  } {
    // the path list is used to control path matching priority
    const pathList: Array<string> = oldPathList || []
    // $flow-disable-line
    const pathMap: Dictionary<RouteRecord> = oldPathMap || Object.create(null)
    // $flow-disable-line
    const nameMap: Dictionary<RouteRecord> = oldNameMap || Object.create(null)
  
    routes.forEach(route => {
      addRouteRecord(pathList, pathMap, nameMap, route)
    })
  
    // ensure wildcard routes are always at the end
    for (let i = 0, l = pathList.length; i < l; i++) {
      if (pathList[i] === '*') {
        pathList.push(pathList.splice(i, 1)[0])
        l--
        i--
      }
    }
  
    return {
      pathList,
      pathMap,
      nameMap
    }
  }
  ```

  遍历我们配置的routes数组，执行`addRouteRecord`方法，目的在将所有的路由都添加到路由映射表中：

  * pathList：包含所有的`path`
  * pathMap：`path`到`RouterRecord`的映射关系表
  * nameMap：`name`到`RouterRecord`的映射关系表

  每个route都定义了一个相应的`RouterRecord`，如果route有children，会递归调用`addRouteRecord`方法，`RouteRecord`是一个树形结构

  ```js
  const record: RouteRecord = {
      path: normalizedPath,
      regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
      components: route.components || { default: route.component },
      instances: {},
      enteredCbs: {},
      name,
      parent,
      matchAs,
      redirect: route.redirect,
      beforeEnter: route.beforeEnter,
      meta: route.meta || {},
      props:
        route.props == null
          ? {}
          : route.components
            ? route.props
            : { default: route.props }
    }
  ```

  这样我们能够快速通过`path`、`name`从对于的映射表中查到对应的`RouteRecord`

* `createMatcher`方法最终，返回一个对象`{match,addRoutes}`，其中`match`方法：

  ```js
  function match (
      raw: RawLocation,
      currentRoute?: Route,
      redirectedFrom?: Location
    ): Route {
      const location = normalizeLocation(raw, currentRoute, false, router)
      const { name } = location
  
      if (name) {
        const record = nameMap[name]
        if (process.env.NODE_ENV !== 'production') {
          warn(record, `Route with name '${name}' does not exist`)
        }
        if (!record) return _createRoute(null, location)
        const paramNames = record.regex.keys
          .filter(key => !key.optional)
          .map(key => key.name)
  
        if (typeof location.params !== 'object') {
          location.params = {}
        }
  
        if (currentRoute && typeof currentRoute.params === 'object') {
          for (const key in currentRoute.params) {
            if (!(key in location.params) && paramNames.indexOf(key) > -1) {
              location.params[key] = currentRoute.params[key]
            }
          }
        }
  
        location.path = fillParams(record.path, location.params, `named route "${name}"`)
        return _createRoute(record, location, redirectedFrom)
      } else if (location.path) {
        location.params = {}
        for (let i = 0; i < pathList.length; i++) {
          const path = pathList[i]
          const record = pathMap[path]
          if (matchRoute(record.regex, location.path, location.params)) {
            return _createRoute(record, location, redirectedFrom)
          }
        }
      }
      // no match
      return _createRoute(null, location)
    }
  ```

  首先调用`normalizeLocation`，当如前所述传入的是url中'#'后的字符串，将其转换成对象格式`{path,query,hash}`

  可以通过`name`和`path`两种情况做匹配：

  - `name`：有 `name` 的情况下就根据 `nameMap` 匹配到 `record`，它就是一个 `RouterRecord` 对象，如果 `record` 不存在，则匹配失败，返回一个空路径；然后拿到 `record` 对应的 `paramNames`，再对比 `currentRoute` 中的 `params`，把交集部分的 `params` 添加到 `location` 中，然后在通过 `fillParams` 方法根据 `record.path` 和 `location.path` 计算出 `location.path`，

  - `path`：通过 `name` 我们可以很快的找到 `record`，但是通过 `path` 并不能，因为我们计算后的 `location.path` 是一个真实路径，而 `record` 中的 `path` 可能会有 `param`，因此需要对所有的 `pathList` 做顺序遍历， 然后通过 `matchRoute` 方法根据 `record.regex`、`location.path`、`location.params` 匹配，因为是顺序遍历，所以我们书写路由配置要注意路径的顺序，因为写在前面的会优先尝试匹配。

  最后都会调用 `_createRoute(record, location, redirectedFrom)` 去生成一条新路径

  ```js
  function _createRoute (
      record: ?RouteRecord,
      location: Location,
      redirectedFrom?: Location
    ): Route {
      if (record && record.redirect) {
        return redirect(record, redirectedFrom || location)
      }
      if (record && record.matchAs) {
        return alias(record, location, record.matchAs)
      }
      return createRoute(record, location, redirectedFrom, router)
    }
  ```

  **如果route配置了redirrct重定向或者alias别名的话会先执行这两个操作**

  先不考虑这两种，执行`createRoute`方法

  ```js
  export function createRoute (
    record: ?RouteRecord,
    location: Location,
    redirectedFrom?: ?Location,
    router?: VueRouter
  ): Route {
    const stringifyQuery = router && router.options.stringifyQuery
  
    let query: any = location.query || {}
    try {
      query = clone(query)
    } catch (e) {}
  
    const route: Route = {
      name: location.name || (record && record.name),
      meta: (record && record.meta) || {},
      path: location.path || '/',
      hash: location.hash || '',
      query,
      params: location.params || {},
      fullPath: getFullPath(location, stringifyQuery),
      matched: record ? formatMatch(record) : []
    }
    if (redirectedFrom) {
      route.redirectedFrom = getFullPath(redirectedFrom, stringifyQuery)
    }
    return Object.freeze(route)
  }
  ```

  创建一条不可以被外界修改的新的route路径，其中最重要的一条属性为`matched:record ? formatMatch(record) : []`

  ```js
  function formatMatch (record: ?RouteRecord): Array<RouteRecord> {
    const res = []
    while (record) {
      res.unshift(record)
      record = record.parent
    }
    return res
  }
  ```

  循环向上找`parent`直到最外层，将其保存在数组中，`matched`记录了一条线路上的所有 `record`，为之后渲染组件提供了依据。

#### 路径切换

匹配完成之后，会将新创建的`route`返回到`transitionTo`的`route`中，接着`transitionTo`方法执行`this.confirmTransition`进行真正的路径切换

```js
transitionTo (location: RawLocation, onComplete?: Function, onAbort?: Function) {
  const route = this.router.match(location, this.current)
  this.confirmTransition(route, () => {
    this.updateRoute(route)
    onComplete && onComplete(route)
    this.ensureURL()

    if (!this.ready) {
      this.ready = true
      this.readyCbs.forEach(cb => { cb(route) })
    }
  }, err => {
    if (onAbort) {
      onAbort(err)
    }
    if (err && !this.ready) {
      this.ready = true
      this.readyErrorCbs.forEach(cb => { cb(err) })
    }
  })
}
```

