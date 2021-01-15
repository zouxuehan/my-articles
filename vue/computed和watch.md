### computed和watch

#### computed

初始化computed`initComputed`

```js
const computedWatcherOptions = { lazy: true }
function initComputed (vm: Component, computed: Object) {
    const watchers = vm._computedWatchers = Object.create(null)
    const isSSR = isServerRendering()

    for (const key in computed) {
        const userDef = computed[key]
        const getter = typeof userDef === 'function' ? userDef : userDef.get
        if (!isSSR) {
            watchers[key] = new Watcher(
                vm,
                getter || noop,
                noop,
                computedWatcherOptions
            )
        }
        if (!(key in vm)) {
            defineComputed(vm, key, userDef)
        } else if (process.env.NODE_ENV !== 'production') {
            if (key in vm.$data) {
                warn(`The computed property "${key}" is already defined in data.`, vm)
            } else if (vm.$options.props && key in vm.$options.props) {
                warn(`The computed property "${key}" is already defined as a prop.`, vm)
            }
        }
    }
}
```

为实例vm添加一个_computedWatchers用于存储所有的computed

遍历computed

1. 获取getter，computed值可以直接写成函数，这种情况就相当于本身就是get，这个getter是传入watcher的回调函数。

2. 为每个computed值实例化一个watcher添加到_computedWatchers中，这里可以得出computed其实也是watcher，只不过它有一个属性`lazy=true`

3. 判断值是否已经存在于data或props中，如果已存在则报错，否则将computed值定义成响应式对象`defineComputed(vm, key, userDef)`

   ```js
   function defineComputed (
   target: any,
    key: string,
    userDef: Object | Function
   ) {
       const shouldCache = !isServerRendering()
       if (typeof userDef === 'function') {
           sharedPropertyDefinition.get = shouldCache
               ? createComputedGetter(key)
           : createGetterInvoker(userDef)
           sharedPropertyDefinition.set = noop
       } else {
           sharedPropertyDefinition.get = userDef.get
               ? shouldCache && userDef.cache !== false
               ? createComputedGetter(key)
           : createGetterInvoker(userDef.get)
           : noop
           sharedPropertyDefinition.set = userDef.set || noop
       }
       Object.defineProperty(target, key, sharedPropertyDefinition)
   }
   ```

   在大多数情况下computed只存在getter`createComputedGetter`

   ```js
   function createComputedGetter (key) {
     return function computedGetter () {
       const watcher = this._computedWatchers && this._computedWatchers[key]
       if (watcher) {
         if (watcher.dirty) {
           watcher.evaluate()
         }
         if (Dep.target) {
           watcher.depend()
         }
         return watcher.value
       }
     }
   }
   ```

   dirty标记computed是否有脏数据，为true则重新计算，否则直接使用缓存

   ```js
   evaluate () {
       this.value = this.get()
       this.dirty = false
   }
   ```

   在之后学watcher的时候会知道computed watcher在实例化的时候不会立即执行this.get，只有在需要计算的时候触发getter的时候执行（在这个过程中computed依赖的值会收集到该watcher的deps中，之后当依赖值改变才会派发其更新）

   这里的Dep.tagert为渲染watcher，执行`watcher.depend()`

   ```js
   depend () {
       let i = this.deps.length
       while (i--) {
           this.deps[i].depend()
       }
   }
   ```

   这里的`this.deps`是依赖该computed watcher的发布者，在DOM中可能并没有用到这些值，所以在render的时候不会触发getter收集依赖，即渲染watcher没有订阅这些值，所以在此处就是为了保证其都被渲染watcher订阅

#### watch

初始化`initWatch`

```js
function initWatch (vm: Component, watch: Object) {
    for (const key in watch) {
        const handler = watch[key]
        if (Array.isArray(handler)) {
            for (let i = 0; i < handler.length; i++) {
                createWatcher(vm, key, handler[i])
            }
        } else {
            createWatcher(vm, key, handler)
        }
    }
}

```

watch可以为数组，即将每一项都创建一个watcher`createWatcher(vm, key, handler)`，注意这里传入watcher的回调为key，之后watcher执行回调的时候就会访问到key触发其getter收集到该watcher订阅者

```js
function createWatcher (
vm: Component,
 expOrFn: string | Function,
 handler: any,
 options?: Object
) {
    if (isPlainObject(handler)) {
        options = handler
        handler = handler.handler
    }
    if (typeof handler === 'string') {
        handler = vm[handler]
    }
    return vm.$watch(expOrFn, handler, options)
}
```

`createWatcher`最终就是获取到handler回调函数执行`$watch`

```js
Vue.prototype.$watch = function (
expOrFn: string | Function,
 cb: any,
 options?: Object
): Function {
    const vm: Component = this
    if (isPlainObject(cb)) {
        return createWatcher(vm, expOrFn, cb, options)
    }
    options = options || {}
    options.user = true
    const watcher = new Watcher(vm, expOrFn, cb, options)
    if (options.immediate) {
        try {
            cb.call(vm, watcher.value)
        } catch (error) {
            handleError(error, vm, `callback for immediate watcher "${watcher.expression}"`)
        }
    }
    return function unwatchFn () {
        watcher.teardown()
    }
}
```

为watch实例化Watcher的时候会传入属性`user=true`，不同的属性代表不同类型的watcher

#### watcher

```js
class Watcher {
  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    this.vm = vm
    if (isRenderWatcher) {
      vm._watcher = this
    }
    vm._watchers.push(this)
    // options
    if (options) {
      this.deep = !!options.deep
      this.user = !!options.user
      this.lazy = !!options.lazy
      this.sync = !!options.sync
      this.before = options.before
    } else {
      this.deep = this.user = this.lazy = this.sync = false
    }
    this.cb = cb
    this.id = ++uid // uid for batching
    this.active = true
    this.dirty = this.lazy // for lazy watchers
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()
    this.expression = process.env.NODE_ENV !== 'production'
      ? expOrFn.toString()
      : ''
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
    }
    this.value = this.lazy
      ? undefined
      : this.get()
  }

```

之前讲依赖收集的时候已经提到过`deps、newDeps、depIds、newDepIds`，主要是用于收集watcher订阅的所有dep

##### get()

除了computed外，其余watcher都会执行get方法

```js
get () {
    pushTarget(this)
    let value
    const vm = this.vm
    try {
      value = this.getter.call(vm, vm)
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // "touch" every property so they are all tracked as
      // dependencies for deep watching
      if (this.deep) {
        traverse(value)
      }
      popTarget()
      this.cleanupDeps()
    }
    return value
  }
```

1. `pushTarget(this)`将当前watcher设置为Dep.target

   ```js
   export function pushTarget (target: ?Watcher) {
     targetStack.push(target)
     Dep.target = target
   }
   ```

   在dep收集依赖的时候执行的是`Dep.target.addDep(this)`，这一操作就会确定dep收集到的是当前的这个watcher

2. `value = this.getter.call(vm, vm)`执行回调

3. `popTarget(this)`将Dep.target恢复到上一个watcher

   ```js
   export function popTarget () {
     targetStack.pop()
     Dep.target = targetStack[targetStack.length - 1]
   }
   ```

##### 不同类型watcher

带有不同属性的watcher代表的不同类型：

1. 渲染watcher

   当所有属性为false的时候就是一个渲染watcher，其最关键的部分就是执行get，从而执行回调函数getter，之所以称为渲染watcher就是因为回调函数执行了渲染操作

2. computed Watcher`this.lazy = true`

   computed Watcher与其他类型的不同在于实例化的时候不执行this.get()

   ```js
   this.value = this.lazy
         ? undefined
         : this.get()
   ```

   以及在依赖的dep派发更新的时候将dirty=true让其在渲染的时候重新计算

   ```js
   update () {
       if (this.lazy) {
           this.dirty = true
       } else if (this.sync) {
           this.run()
       } else {
           queueWatcher(this)
       }
   }
   ```

3. user Watcher `this.user = true`

   用户写的watch的主要区别在于会处理一下错误

   ```js
   if (this.user) {
       handleError(e, vm, `getter for watcher "${this.expression}"`)
   }
   ```

4. deep Watcher `this.deep = true`

   当你为一个对象手写一个watch的时候，之前有提到会为key收集到watcher，但是如果key为一个对象，而修改key的属性时

   ```js
   var vm = new Vue({
       data() {
           a: {
               b: 1
           }
       },
       watch: {
           a: {
               handler(newVal) {
                   console.log(newVal)
               }
           }
       }
   })
   vm.a.b = 2
   ```

   这时发布者是a.b，但是它并没有收集到这个watcher，所以不会执行回调，Vue解决这个问题的办法就是添加`deep:true`的属性

   ```js
   watch: {
     a: {
       deep: true,
       handler(newVal) {
         console.log(newVal)
       }
     }
   }
   ```

   这时实例化该watcher执行get的时候会执行`traverse`：

   ```js
   if (this.deep) {
       traverse(value)
   }
   
   function traverse (val: any) {
     _traverse(val, seenObjects)
     seenObjects.clear()
   }
   
   function _traverse (val: any, seen: SimpleSet) {
     let i, keys
     const isA = Array.isArray(val)
     if ((!isA && !isObject(val)) || Object.isFrozen(val) || val instanceof VNode) {
       return
     }
     if (val.__ob__) {
       const depId = val.__ob__.dep.id
       if (seen.has(depId)) {
         return
       }
       seen.add(depId)
     }
     if (isA) {
       i = val.length
       while (i--) _traverse(val[i], seen)
     } else {
       keys = Object.keys(val)
       i = keys.length
       while (i--) _traverse(val[keys[i]], seen)
     }
   }
   ```

   _traverse方法就是执行递归实现对象的深度遍历，实际上就会触发对象所有属性的getter，这样所有属性都收集到了该watcher

   

