### Vue实例挂载

```js
if (vm.$options.el) {
    vm.$mount(vm.$options.el)
}
```

$mount挂载是和平台、构建方式相关的，在整体学习中会知道最终渲染都是要通过render方法的，所以当用户使用的是<template> 时需要将其编译，而此过程就发生在$mount方法中，但编译是一个非常复杂的过程，将另外展开

```js
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}
```

最终将调用`mountComponent`函数

#### mountComponent

```js
function mountComponent (
vm: Component,
 el: ?Element,
 hydrating?: boolean
): Component {
    vm.$el = el
    if (!vm.$options.render) {
        vm.$options.render = createEmptyVNode
    }
    callHook(vm, 'beforeMount')

    let updateComponent

    updateComponent = () => {
        vm._update(vm._render(), hydrating)
    }

    new Watcher(vm, updateComponent, noop, {
        before () {
            if (vm._isMounted && !vm._isDestroyed) {
                callHook(vm, 'beforeUpdate')
            }
        }
    }, true /* isRenderWatcher */)
    hydrating = false

    if (vm.$vnode == null) {
        vm._isMounted = true
        callHook(vm, 'mounted')
    }
    return vm
}
```

首先执行了生命周期钩子beforeMount，接着设置了一个函数`updateComponent`，该函数作为传入Watcher的回调函数，创建一个Watcher实例，这个就是渲染watcher，在讲watcher的时候知道了它将执行该回调`updateComponent`，实际在此方法中调用 `vm._render` 方法先生成虚拟DOM，传入 `vm._update`渲染/更新真实 DOM

#### 虚拟DOM

虚拟DOM是Vue设计的一个重点，当我们使用前端三件套写页面的时候，经常会用js来频繁操作DOM节点，这也就使得页面会发生重绘/回流，产生较大的性能问题

虚拟DOM在vue中用类VNode来表示，是用js对象来抽象描述DOM节点，它映射到真实DOM需要经历create->diff->patch。

#### render

```js
Vue.prototype._render = function (): VNode {
    const vm: Component = this
    const { render, _parentVnode } = vm.$options

    if (_parentVnode) {
        vm.$scopedSlots = normalizeScopedSlots(
            _parentVnode.data.scopedSlots,
            vm.$slots,
            vm.$scopedSlots
        )
    }

    vm.$vnode = _parentVnode
    // render self
    let vnode
    try {
        currentRenderingInstance = vm
        vnode = render.call(vm._renderProxy, vm.$createElement)
    } catch (e) {
        handleError(e, vm, `render`)
        vnode = vm._vnode
    } finally {
        currentRenderingInstance = null
    }

    if (Array.isArray(vnode) && vnode.length === 1) {
        vnode = vnode[0]
    }

    if (!(vnode instanceof VNode)) {
        if (process.env.NODE_ENV !== 'production' && Array.isArray(vnode)) {
            warn(
                'Multiple root nodes returned from render function. Render function ' +
                'should return a single root node.',
                vm
            )
        }
        vnode = createEmptyVNode()
    }
    // set parent
    vnode.parent = _parentVnode
    return vnode
}
```

首先判断是否存在_parentVnode，`_parentVnode`来自vm.$options，来自初始化组件实例合并配置时，然后格式化slot

在vue中写的render函数如下：

```js
render: function (createElement) {
    return createElement(
      'h' + this.level,   // 标签名称
      this.$slots.default // 子节点数组
    )
  }
```

执行`vnode = render.call(vm._renderProxy, vm.$createElement)`，最终返回的是`vm.$createElement()`的执行结果。

```js
vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)

function createElement (
context: Component,
 tag: any,
 data: any,
 children: any,
 normalizationType: any,
 alwaysNormalize: boolean
): VNode | Array<VNode> {
    if (Array.isArray(data) || isPrimitive(data)) {
        normalizationType = children
        children = data
        data = undefined
    }
    if (isTrue(alwaysNormalize)) {
        normalizationType = ALWAYS_NORMALIZE
    }
    return _createElement(context, tag, data, children, normalizationType)
}
```

关于传入的参数，结合vue官网中定义的createElement参数更好来理解：

```js
createElement(
  // {String | Object | Function}
  // 一个 HTML 标签名、组件选项对象，或者
  // resolve 了上述任何一种的一个 async 函数。必填项。
  'div',

  // {Object}
  // 一个与模板中 attribute 对应的数据对象。可选。
  {
    // (详情见下一节)
  },

  // {String | Array}
  // 子级虚拟节点 (VNodes)，由 `createElement()` 构建而成，
  // 也可以使用字符串来生成“文本虚拟节点”。可选。
  [
    '先写一些文字',
    createElement('h1', '一则头条'),
    createElement(MyComponent, {
      props: {
        someProp: 'foobar'
      }
    })
  ]
)
```

对照来看，`tag`即表示为标签/组件，`data`应该为模板中attribute 对应的数据对象，`children`为子节点，`normalizationType`用来表示是编译得到的render函数还是用户自己写的

在这一步的处理中，如果传入的第二个参数不是对象即省略了attribute数据对象，此时的第二个参数就应该为`children`，实际的`data=undefined`，vue有许多地方通过这样的封装使得用户在传递参数的时候更为灵活。

#### createElement

```js
function _createElement (
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
): VNode | Array<VNode> {
  // object syntax in v-bind
  if (isDef(data) && isDef(data.is)) {
    tag = data.is
  }
  if (!tag) {
    // in case of component :is set to falsy value
    return createEmptyVNode()
  }
  // support single function children as default scoped slot
  if (Array.isArray(children) &&
    typeof children[0] === 'function'
  ) {
    data = data || {}
    data.scopedSlots = { default: children[0] }
    children.length = 0
  }
  if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
  }
  let vnode, ns
  if (typeof tag === 'string') {
    let Ctor
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    if (config.isReservedTag(tag)) {
      vnode = new VNode(
        config.parsePlatformTagName(tag), data, children,
        undefined, undefined, context
      )
    } else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // component
      vnode = createComponent(Ctor, data, context, children, tag)
    } else {
      // unknown or unlisted namespaced elements
      // check at runtime because it may get assigned a namespace when its
      // parent normalizes children
      vnode = new VNode(
        tag, data, children,
        undefined, undefined, context
      )
    }
  } else {
    // direct component options / constructor
    vnode = createComponent(tag, data, context, children)
  }
  if (Array.isArray(vnode)) {
    return vnode
  } else if (isDef(vnode)) {
    if (isDef(ns)) applyNS(vnode, ns)
    if (isDef(data)) registerDeepBindings(data)
    return vnode
  } else {
    return createEmptyVNode()
  }
}
```

经过封装处理，传入`_createElement`的参数一定是规范的了

1. 首先对data.is做了判断，该情况对应的模板语法为`<component v-bind:is="currentTabComponent"></component>`
2. **接着对scopedSlots进行处理？？？**
3. children规范化
4. 创建vnode

children规范化和创建vnode是`createElement`的重点，展开学习：

##### children规范化

创建vnode的时候要求`children?: ?Array<VNode>`，即children是由Vnode组成的数组，所以需要将children每一项都创建成vnode

```js
function simpleNormalizeChildren (children: any) {
  for (let i = 0; i < children.length; i++) {
    if (Array.isArray(children[i])) {
      return Array.prototype.concat.apply([], children)
    }
  }
  return children
}
function normalizeChildren (children: any): ?Array<VNode> {
  return isPrimitive(children)
    ? [createTextVNode(children)]
    : Array.isArray(children)
      ? normalizeArrayChildren(children)
      : undefined
}
```

如果是编译得到的render函数，其children就已经都是vnode了，则只需要简单处理将其扁平化，使得数组深度只有一层**???**

如果是用户手写的render函数，当children是基础类型则只要创建一个文本节点vnode，如果是数组执行`normalizeArrayChildren`

```js
function normalizeArrayChildren (children: any, nestedIndex?: string): Array<VNode> {
  const res = []
  let i, c, lastIndex, last
  for (i = 0; i < children.length; i++) {
    c = children[i]
    if (isUndef(c) || typeof c === 'boolean') continue
    lastIndex = res.length - 1
    last = res[lastIndex]
    //  nested
    if (Array.isArray(c)) {
      if (c.length > 0) {
        c = normalizeArrayChildren(c, `${nestedIndex || ''}_${i}`)
        // merge adjacent text nodes
        if (isTextNode(c[0]) && isTextNode(last)) {
          res[lastIndex] = createTextVNode(last.text + (c[0]: any).text)
          c.shift()
        }
        res.push.apply(res, c)
      }
    } else if (isPrimitive(c)) {
      if (isTextNode(last)) {
        // merge adjacent text nodes
        // this is necessary for SSR hydration because text nodes are
        // essentially merged when rendered to HTML strings
        res[lastIndex] = createTextVNode(last.text + c)
      } else if (c !== '') {
        // convert primitive to vnode
        res.push(createTextVNode(c))
      }
    } else {
      if (isTextNode(c) && isTextNode(last)) {
        // merge adjacent text nodes
        res[lastIndex] = createTextVNode(last.text + c.text)
      } else {
        // default key for nested array children (likely generated by v-for)
        if (isTrue(children._isVList) &&
          isDef(c.tag) &&
          isUndef(c.key) &&
          isDef(nestedIndex)) {
          c.key = `__vlist${nestedIndex}_${i}__`
        }
        res.push(c)
      }
    }
  }
  return res
}
```

遍历children

1. 如果**该项为数组？？？**，则使用递归将其扁平化，目的也是得到深度为1的数组
2. 如果为基础类型，则创建文本节点`createTextVNode`
3. 否则已经是vnode，**如果 `children` 是一个列表并且列表还存在嵌套的情况，则根据 `nestedIndex` 去更新它的 key？？？**

并且在这些情况中都做了一个处理：如果存在两个连续的文本节点，会将其合并成一个。

经过children规范化处理之后，children最终变成一个深度为1，类型为vnode的数组。

##### VNode创建

如果tag是字符串且是内置节点，则直接创建普通vnode，如果是组件名或组件类型，则调用`createComponent`创建组件类型vnode。

#### update

render生成vnode之后传入update

```js
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    const vm: Component = this
    const prevEl = vm.$el
    const prevVnode = vm._vnode
    const restoreActiveInstance = setActiveInstance(vm)
    vm._vnode = vnode
    // Vue.prototype.__patch__ is injected in entry points
    // based on the rendering backend used.
    if (!prevVnode) {
      // initial render
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    restoreActiveInstance()
    // update __vue__ reference
    if (prevEl) {
      prevEl.__vue__ = null
    }
    if (vm.$el) {
      vm.$el.__vue__ = vm
    }
    // if parent is an HOC, update its $el as well
    if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
      vm.$parent.$el = vm.$el
    }
    // updated hook is called by the scheduler to ensure that children are
    // updated in a parent's updated hook.
  }
```

分为两种情况，初次渲染和更新，都执行`__patch_ _`方法，只是传入参数不同，该方法在不同的平台是不一样的，在web端定义为`Vue.prototype.__patch__ = inBrowser ? patch : noop`

patch方法`const patch: Function = createPatchFunction({ nodeOps, modules })`实际为`createPatchFunction`返回的`patch`函数，该处使用了函数柯里化的技巧，`nodeOps`封装了DOM操作方法，`modules` 定义了一些模块的钩子函数的实现，在不同平台有不同的nodeOps和modules，使用柯里化之后，将差异化参数提前固化，就不需要每次调用patch的时候都传递这些参数。

#### patch

```js
function createPatchFunction (backend) {
    return function patch (oldVnode, vnode, hydrating, removeOnly) {
    if (isUndef(vnode)) {
      if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
      return
    }

    let isInitialPatch = false
    const insertedVnodeQueue = []

    if (isUndef(oldVnode)) {
      // empty mount (likely as component), create new root element
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue)
    } else {
      const isRealElement = isDef(oldVnode.nodeType)
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        // patch existing root node
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
      } else {
        if (isRealElement) {
          if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
            oldVnode.removeAttribute(SSR_ATTR)
            hydrating = true
          }
          if (isTrue(hydrating)) {
            if (hydrate(oldVnode, vnode, insertedVnodeQueue)) {
              invokeInsertHook(vnode, insertedVnodeQueue, true)
              return oldVnode
            } else if (process.env.NODE_ENV !== 'production') {
              warn()
            }
          }
          // either not server-rendered, or hydration failed.
          // create an empty node and replace it
          oldVnode = emptyNodeAt(oldVnode)
        }

        // replacing existing element
        const oldElm = oldVnode.elm
        const parentElm = nodeOps.parentNode(oldElm)

        // create new node
        createElm(
          vnode,
          insertedVnodeQueue,
          // extremely rare edge case: do not insert if old element is in a
          // leaving transition. Only happens when combining transition +
          // keep-alive + HOCs. (#4590)
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        )

        // update parent placeholder node element, recursively
        if (isDef(vnode.parent)) {
          let ancestor = vnode.parent
          const patchable = isPatchable(vnode)
          while (ancestor) {
            for (let i = 0; i < cbs.destroy.length; ++i) {
              cbs.destroy[i](ancestor)
            }
            ancestor.elm = vnode.elm
            if (patchable) {
              for (let i = 0; i < cbs.create.length; ++i) {
                cbs.create[i](emptyNode, ancestor)
              }
              // #6513
              // invoke insert hooks that may have been merged by create hooks.
              // e.g. for directives that uses the "inserted" hook.
              const insert = ancestor.data.hook.insert
              if (insert.merged) {
                // start at index 1 to avoid re-invoking component mounted hook
                for (let i = 1; i < insert.fns.length; i++) {
                  insert.fns[i]()
                }
              }
            } else {
              registerRef(ancestor)
            }
            ancestor = ancestor.parent
          }
        }

        // destroy old node
        if (isDef(parentElm)) {
          removeVnodes([oldVnode], 0, 0)
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode)
        }
      }
    }
}
```

patch方法很复杂分为多种情况，这里只讨论vue实例挂载的情况：

```js
var app = new Vue({
  el: '#app',
  render: function (createElement) {
    return createElement('div', {
      attrs: {
        id: 'app'
      },
    }, this.message)
  },
  data: {
    message: 'Hello Vue!'
  }
})
```

初次渲染的时候，传入的参数为`vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)`

此时`isRealElement = isDef(*oldVnode*.nodeType)`为true，执行else逻辑，首先会执行`oldVnode = emptyNodeAt(oldVnode)`将真实DOM转为vnode

```js
function emptyNodeAt (elm) {
    return new VNode(nodeOps.tagName(elm).toLowerCase(), {}, [], undefined, elm)
  }
```

接着为最重要的`createElm`，它的作用就是将vnode创建成真实DOM插入到父节点上

##### createElm

```js
createElm(
    vnode,
    insertedVnodeQueue,
    // extremely rare edge case: do not insert if old element is in a
    // leaving transition. Only happens when combining transition +
    // keep-alive + HOCs. (#4590)
    oldElm._leaveCb ? null : parentElm,
    nodeOps.nextSibling(oldElm)
)
function createElm (
vnode,
 insertedVnodeQueue,
 parentElm,
 refElm,
 nested,
 ownerArray,
 index
) {
    if (isDef(vnode.elm) && isDef(ownerArray)) {
        vnode = ownerArray[index] = cloneVNode(vnode)
    }

    vnode.isRootInsert = !nested 
    if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
        return
    }

    const data = vnode.data
    const children = vnode.children
    const tag = vnode.tag
    if (isDef(tag)) {
        vnode.elm = vnode.ns
            ? nodeOps.createElementNS(vnode.ns, tag)
        : nodeOps.createElement(tag, vnode)
        setScope(vnode)

        /* istanbul ignore if */
        if (__WEEX__) {
           
        } else {
            createChildren(vnode, children, insertedVnodeQueue)
            if (isDef(data)) {
                invokeCreateHooks(vnode, insertedVnodeQueue)
            }
            insert(parentElm, vnode.elm, refElm)
        }

        if (process.env.NODE_ENV !== 'production' && data && data.pre) {
            creatingElmInVPre--
        }
    } else if (isTrue(vnode.isComment)) {
        vnode.elm = nodeOps.createComment(vnode.text)
        insert(parentElm, vnode.elm, refElm)
    } else {
        vnode.elm = nodeOps.createTextNode(vnode.text)
        insert(parentElm, vnode.elm, refElm)
    }
}
```

1. `createComponent(vnode, insertedVnodeQueue, parentElm, refElm)`只有在`isDef(vnode.componentInstance)`为ture的时候返回true，该方法的目的是创建子组件，这部分在组件的时候再展开，该处为false

2. 接着判断tag，调用平台DOM操作方法创建一个占位符元素。

3. web端`createChildren(vnode, children, insertedVnodeQueue)`，深度遍历children执行createElm

   ```js
   function createChildren (vnode, children, insertedVnodeQueue) {
       if (Array.isArray(children)) {
         for (let i = 0; i < children.length; ++i) {
           createElm(children[i], insertedVnodeQueue, vnode.elm, null, true, children, i)
         }
       } else if (isPrimitive(vnode.text)) {
         nodeOps.appendChild(vnode.elm, nodeOps.createTextNode(String(vnode.text)))
       }
     }
   ```

4. 调用`invokeCreateHooks`，该方法执行所有组件的create钩子，在之后组件中展开

5. 最后将DOM插入父节点中`insert(parentElm, vnode.elm, refElm)`

   ```js
   function insert (parent, elm, ref) {
       if (isDef(parent)) {
         if (isDef(ref)) {
           if (nodeOps.parentNode(ref) === parent) {
             nodeOps.insertBefore(parent, elm, ref)
           }
         } else {
           nodeOps.appendChild(parent, elm)
         }
       }
     }
   ```

因为是递归调用`createElm`，所以子元素先调用`insert`，即整个树节点的插入顺序是先子后父

执行完`createElm`，最后更新父占位元素，移除oldVnode，执行insert钩子