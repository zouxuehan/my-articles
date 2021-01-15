### 响应式原理

响应式原理的核心是`Object.defineProperty`为对象设置getter和setter

#### initData

initData的核心内容主要是代理proxy和监测observe，在此之前还要获取组件data函数的返回对象

#### 代理proxy

遍历data，逐一进行代理`proxy(vm,'_data', key)`

```js
function proxy (target: Object, sourceKey: string, key: string) {
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

proxy代理的实现是通过getter，setter将this.xxx设置到this._data.xxx上，相当于设置拦截

#### 监测observe

`observe(data, true /* asRootData */)`监测data，其主要目的就是给**非VNode的对象类型创建一个Observer**（监测对象）`ob = new Observer(data)`

##### Observer

```js
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

  constructor (value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      if (hasProto) {
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }
...
}
```

Observer构造函数重点为this实例化一个**Dep**实例（可理解为真正的被观察者），然后将自身实例添加到数据对象value. _ _ob _ _属性上，然后判断是否为数组分成了两种情况

* 非数组

  在第一次执行observe的数据对象是选项data，肯定是一个对象，这时执行`this.walk(value)`

  ```js
  walk (obj: Object) {
      const keys = Object.keys(obj)
      for (let i = 0; i < keys.length; i++) {
        defineReactive(obj, keys[i])
      }
    }
  ```

  walk方法遍历所有的数据对象属性，逐一执行响应式操作**defineReactive**

* 数组

  遍历执行**defineReactive**的时候如果遇到对象类型会再次进行observe，这时就会出现数组的情况执行`this.observeArray(value)`

  ```js
  observeArray (items: Array<any>) {
      for (let i = 0, l = items.length; i < l; i++) {
        observe(items[i])
      }
  }
  ```

  observeArray方法也是遍历所有项进行observe监测，这里就会产生一个问题，如果item不是对象会直接返回，不会进行响应式操作，**因此如果你直接操作数组的单独项不是响应式的**

  ```js
  var vm = new Vue({
    data: {
      items: ['a', 'b', 'c']
    }
  })
  vm.items[1] = 'x' // 不是响应性的
  vm.items.length = 2 // 不是响应性的
  ```

  解决的办法见最后

##### defineReactive

1. defineReactive首先初始化一个Dep实例

2. 接着将属性再进行observe监测`childOb = !shallow && observe(val)`，这一步的作用是利用递归将所有的对象属性都进行监测，这样无论data的数据结构多复杂，所有对象的子属性都会变成响应式的；

3. **响应式的核心则是`Object.defineProperty`**，为属性值都设置getter和setter

#### getter依赖收集

```js
get: function reactiveGetter () {
    const value = getter ? getter.call(obj) : val
    if (Dep.target) {
        dep.depend()
        if (childOb) {
            childOb.dep.depend()
            if (Array.isArray(value)) {
                dependArray(value)
            }
        }
    }
    return value
}
```

这里有一个Dep.target，关于这个值要涉及到**watcher**，在实例化渲染watcher时会`pushTarget(this)`，此时Dep.target就是渲染watcher，渲染watcher再执行update回调前会render生成虚拟DOM，render过程中就会读取所有的需要渲染的值，会触发这些值的getter

##### Dep

```js
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;
  constructor () {
    this.id = uid++
    this.subs = []
  }
  addSub (sub: Watcher) {
      //收集订阅者
      this.subs.push(sub)
  } 
  removeSub (sub: Watcher) {
      //移除订阅者
      remove(this.subs, sub)
  } 
  depend () {
      //依赖
      if (Dep.target) {
        Dep.target.addDep(this)
    }
  }
  notify () {}//派发更新
}
```

这篇有很多地方用到了Dep，它可以理解成一个发布者，每个dep实例有id和subs，**subs用来收集其所有的订阅者watcher**

##### Watcher

```js
class Watcher {
  id: number;
  deps: Array<Dep>;
  newDeps: Array<Dep>;
  depIds: SimpleSet;
  newDepIds: SimpleSet;
  getter: Function;
  value: any;
  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    ...
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()
   ...
  }
  ...
  addDep (dep: Dep) {//收集发布者
      const id = dep.id
      if (!this.newDepIds.has(id)) {
          this.newDepIds.add(id)
          this.newDeps.push(dep)
          if (!this.depIds.has(id)) {
              dep.addSub(this)
          }
      }
  }
  cleanupDeps () {}//清空newDeps
  update () {}//更新
  ...
}
```

这里的watcher是订阅者，以上代码只列出了与响应式相关的内容，其中每个watcher都有与dep相关的属性`deps,newDeps,depIds,newDepIds`，**deps用来收集所有的发布者**，其中newDeps是为了做优化。

缺图一张

#### setter派发更新

```js
set: function reactiveSetter (newVal) {
    const value = getter ? getter.call(obj) : val
    /* eslint-disable no-self-compare */
    if (newVal === value || (newVal !== newVal && value !== value)) {
        return
    }
    /* eslint-enable no-self-compare */
    if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
    }
    // #7981: for accessor properties without setter
    if (getter && !setter) return
    if (setter) {
        setter.call(obj, newVal)
    } else {
        val = newVal
    }
    childOb = !shallow && observe(newVal)
    dep.notify()
}
```

如果新旧值相等直接返回，否则值改变并`observe(newVal)`设置为响应式，然后进行派发`dep.notify()`

```js
notify () {
    const subs = this.subs.slice()
    for (let i = 0, l = subs.length; i < l; i++) {
        subs[i].update()
    }
}
```

通知其依赖的所有watcher进行更新`subs[i].update()`

```js
update () {
    /* istanbul ignore else */
    if (this.lazy) {
        this.dirty = true
    } else if (this.sync) {
        this.run()
    } else {
        queueWatcher(this)
    }
}
```

update针对不同的watcher类型分了三种情况（Watch篇），这里最重要的就是`queueWatcher(this)`**异步更新队列**。

#### **异步更新队列**

Vue官网中提到：

> Vue 在更新 DOM 时是**异步**执行的。只要侦听到数据变化，Vue 将开启一个队列，并缓冲在同一事件循环中发生的所有数据变更。如果同一个 watcher 被多次触发，只会被推入到队列中一次。这种在缓冲时去除重复数据对于避免不必要的计算和 DOM 操作是非常重要的。

```js
function queueWatcher (watcher: Watcher) {
    const id = watcher.id
    if (has[id] == null) {
        has[id] = true
        if (!flushing) {
            queue.push(watcher)
        } else {
            let i = queue.length - 1
            while (i > index && queue[i].id > watcher.id) {
                i--
            }
            queue.splice(i + 1, 0, watcher)
        }
        // queue the flush
        if (!waiting) {
            waiting = true
            nextTick(flushSchedulerQueue)
        }
    }
}
```

`let has: { [*key*: number]: ?true } = {}`has对象确保watcher只被添加到队列中一次，

flushing表示是否已刷新（开始）队列，已刷新的情况发生在nextTick异步回调执行`flushSchedulerQueue`的时候，当还未刷新队列时将watcher推入队列。

`waiting` 保证对 `nextTick(flushSchedulerQueue)` 的调用逻辑只有一次。

##### nextTick

```js
function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    timerFunc()
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```

nextTick将回调函数异步执行，callbacks收集回调，`pending`在第一次执行nextTick的时候决定异步操作的方式`timerFunc()`

**timerFunc**的目的是进行能力检测，来决定用什么方式来执行回调函数`flushCallbacks`，根据性能，优先级顺序为 **Promise->MutationObserver->setImmediate->setTimeout**

最终异步回调执行的是`flushCallbacks`

```js
function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}
```

遍历执行callbacks队列中的回调函数，除去我们自己写的`this.$nextTick(()=>{})`中的回调，Vue实现异步DOM更新的操作回到**`flushSchedulerQueue`**

##### flushSchedulerQueue

```js
function flushSchedulerQueue () {
    currentFlushTimestamp = getNow()
    flushing = true
    let watcher, id
	//watcher队列排序
    queue.sort((a, b) => a.id - b.id)
	//watcher队列遍历执行更新
    for (index = 0; index < queue.length; index++) {
        watcher = queue[index]
        if (watcher.before) {
            watcher.before()
        }
        id = watcher.id
        has[id] = null
        watcher.run()
    }
    const activatedQueue = activatedChildren.slice()
    const updatedQueue = queue.slice()
	// 重置队列状态
    resetSchedulerState()
	// 执行更新钩子函数
    callActivatedHooks(activatedQueue)
    callUpdatedHooks(updatedQueue)

    if (devtools && config.devtools) {
        devtools.emit('flush')
    }
}
```

1. 首先`flushing = true`标记已经开始刷新队列了

2. `queue.sort((a, b) => a.id - b.id)`将队列中的watcher按照id从小到大排列，先创建的先更新（父组件优于子，自定义watcher优于渲染watcher）

3. 接下来遍历队列执行`watcher.run`，这里注意一点，在这个过程中用户可能添加新的watcher，再次执行`queueWatcher`

   ```js
   if (!flushing) {
       queue.push(watcher)
   } else {
       let i = queue.length - 1
       while (i > index && queue[i].id > watcher.id) {
           i--
       }
       queue.splice(i + 1, 0, watcher)
   }
   ```

   当flushing为true时不是简单的push到队列最后，而是根据id大小添加到相应位置（因为只会执行一次`flushSchedulerQueue`排序一次，在其中新添加的watcher也应该按顺序插入）

4. 重置队列状态`resetSchedulerState`，将一些相关变量恢复初始值(has、waiting、flushing...)

#### 检测变化的注意事项

在Vue官网中提到：由于 JavaScript 的限制，Vue **不能检测**数组和对象的变化。

##### 对于对象

`observe`方法为对象实例化一个Observer，遍历对象属性执行defineReactive添加getter和setter变成响应式，但是为**对象新增的属性不会执行defineReactive即为非响应式的**。

vue为解决这个问题新增了`Vue.set(target,key,val)`方法

```js
function set (target: Array<any> | Object, key: any, val: any): any {
    if (Array.isArray(target) && isValidArrayIndex(key)) {
        target.length = Math.max(target.length, key)
        target.splice(key, 1, val)
        return val
    }
    if (key in target && !(key in Object.prototype)) {
        target[key] = val
        return val
    }
    const ob = (target: any).__ob__
    if (!ob) {
        target[key] = val
        return val
    }
    defineReactive(ob.value, key, val)
    ob.dep.notify()
    return val
}
```

1. 首先做了校验key是否已存在，已存在的情况对于对象直接赋值就会触发setter，对于数组执行target.splice(key, 1, val)
2. 接着获取target._ _ob__，在实例化Observer的时候会将该实例添加到`__ob__`上，所以响应式对象会有该属性，如果不是响应式对象则直接赋值
3. 如果是响应式对象，执行`defineReactive(ob.value, key, val)`将该属性变成响应式
4. 最后手动派发更新

##### 对于数组

之前提到数组在实例化Observer的时候执行`observeArray`，该方法遍历数组将每一项执行observe监测，但是observe只对非VNode的对象类型做响应式操作，即数组中的基础类型值都是非响应式的

```js
var vm = new Vue({
  data: {
    items: ['a', 'b', 'c']
  }
})
vm.items[1] = 'x' // 不是响应性的
vm.items.length = 2 // 不是响应性的
```

Vue给出的解决办法`vm.items.splice(indexOfItem, 1, newValue)`实际上是**重写了会改变数组本身的原型方法**：

```js
const arrayProto = Array.prototype
export const arrayMethods = Object.create(arrayProto)
const methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]
methodsToPatch.forEach(function (method) {
  // cache original method
  const original = arrayProto[method]
  def(arrayMethods, method, function mutator (...args) {
    const result = original.apply(this, args)
    const ob = this.__ob__
    let inserted
    switch (method) {
      case 'push':
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    if (inserted) ob.observeArray(inserted)
    // notify change
    ob.dep.notify()
    return result
  })
})
```

1. 重写后先执行它们原有的逻辑，
2. 对能增加数组长度的 `push、unshift、splice` 方法做了判断，获取到插入的值
3. 把新添加的值变成一个响应式对象
4. 最后手动派发更新