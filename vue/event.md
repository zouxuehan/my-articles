### event

在编译阶段，`addHandler`先将对事件修饰符做处理，`.native`判断是原生事件还是普通事件，最后将`name`进行事件归类，并将回调函数保留到对应事件中。

在codegen阶段，`genHandler`方法处理事件修饰符：

> * `.stop`：`$event.stopPropagation();`立即阻止事件流在 DOM 结构中传播，取消后续的事件捕获或冒泡。
> * `.prevent`：`$event.preventDefault();`用于阻止特定事件的默认动作。
> * `.self`：`if($event.target !== $event.currentTarget)return null;`只当在 event.target 是当前元素自身时触发处理函数

#### 原生DOM事件

在patch阶段，经常提到各个`module`的钩子函数，这些 `module` 的钩子函数就是用于完成设置DOM 元素相关的属性、样式、事件等的。

在patch的创建和更新阶段都会执行`updateDOMListeners`：

* 该方法首先调用`normalizeEvents`对`v-model`做相应处理

* 接着调用 `updateListeners(on, oldOn, add, remove, vnode.context)` ：遍历`on`首先处理`once,capture,passive`修饰符

  > ![image-20210119100419240](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210119100419240.png)

  再调用`add`方法添加事件处理程序

  ```js
  function add (
    name: string,
    handler: Function,
    capture: boolean,
    passive: boolean
  ) {
    target.addEventListener(
      name,
      handler,
      supportsPassive
        ? { capture, passive }
        : capture
    )
  }
  ```

  更新阶段还要遍历`oldOn`调用`remove`移除事件处理程序

  ```js
  function remove (
    name: string,
    handler: Function,
    capture: boolean,
    _target?: HTMLElement
  ) {
    (_target || target).removeEventListener(
      name,
      handler._wrapper || handler,
      capture
    )
  }
  ```

#### 自定义事件

自定义事件是作用在组件上的，用于父子组件

在render阶段创建组件节点`createComponent`时有提到会提取子组件监听器：

```js
export function createComponent (
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  // ...
  const listeners = data.on
  
  data.on = data.nativeOn
  
  // ...
  const name = Ctor.options.name || tag
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children },
    asyncFactory
  )

  return vnode
}
```

将`data.on`赋值给了`listeners`，将`data.nativeOn`赋给`data.on`，这样在执行上述说的钩子函数时，原生事件就会添加到当前组件的事件处理程序中，而对于自定义事件而言，`listeners`被传入`vnode`，在实例化子组件的时候才被处理

之前在`_init`合并配置的组件情况中提到：父组件传入的监听器`listeners`被赋值给`vm.$options._parentListeners`，再执行`initEvents`的时候进行处理

```js
export function initEvents (vm: Component) {
  vm._events = Object.create(null)
  vm._hasHookEvent = false
  // init parent attached events
  const listeners = vm.$options._parentListeners
  if (listeners) {
    updateComponentListeners(vm, listeners)
  }
}
export function updateComponentListeners (
  vm: Component,
  listeners: Object,
  oldListeners: ?Object
) {
  target = vm
  updateListeners(listeners, oldListeners || {}, add, remove, createOnceHandler, vm)
  target = undefined
}
```

也是执行`updateListeners`，但是该处传入的`add`和`remove`函数不一样

```js
function add (event, fn) {
  target.$on(event, fn)
}
function remove (event, fn) {
  target.$off(event, fn)
}
```

`add`函数实际调用`$on`，该方法就是将该事件的回调函数存储到`vm._events[event]`中

```js
Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {
    const vm: Component = this
    if (Array.isArray(event)) {
      for (let i = 0, l = event.length; i < l; i++) {
        vm.$on(event[i], fn)
      }
    } else {
      (vm._events[event] || (vm._events[event] = [])).push(fn)
      // optimize hook:event cost by using a boolean flag marked at registration
      // instead of a hash lookup
      if (hookRE.test(event)) {
        vm._hasHookEvent = true
      }
    }
    return vm
  }
```

当你触发子组件的事件时会执行`this.$emit(eventName,[...args])`，该方法就是根据事件名找到所有存储的回调函数，然后遍历执行它们

```js
Vue.prototype.$emit = function (event: string): Component {
    const vm: Component = this
    let cbs = vm._events[event]
    if (cbs) {
      cbs = cbs.length > 1 ? toArray(cbs) : cbs
      const args = toArray(arguments, 1)
      const info = `event handler for "${event}"`
      for (let i = 0, l = cbs.length; i < l; i++) {
        invokeWithErrorHandling(cbs[i], vm, args, vm, info)
      }
    }
    return vm
  }
```





