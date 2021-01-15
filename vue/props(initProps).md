### props

​		在`initState`中首先初始化了props，在此之前合并配置`mergeOption`时对props进行了规范化`normalizeProps`

​		因为语法规定了props可以写成数组或对象，并且对象允许配置高级选项，如type、default、required、validator。

​		为了统一`normalizeProps`最终都将props规范化为对象的形式：

```js
props: ['name', 'nick-name'] ==>
options.props = {
  name: { type: null },
  nickName: { type: null }
}
```

`initProps`的核心步骤为以下三步，遍历props逐一进行：

* 校验`validateProp(key, propsOptions, propsData, vm)`
* 响应式`defineReactive(props, key, value)`
* 代理`proxy(vm, '_props', key)`

#### 校验

主要校验3步：

* **处理boolean类型值：**

  `absent && !hasOwn(prop, 'default')`如果父组件未传值且无默认值则为false，

  `value === '' || value === hyphenate(key)`如果父组件传值为空或为连字符形式(eg:`<student nick-name/>`/`<student nick-name="nick-name"/>`)则为true。

* 处理默认数据：

  当父组件未传值时，获取props的default值`getPropDefaultValue(vm, prop, key)`

  如果没有default值则返回undefined；

  如果上一次组件渲染父组件传递的 `prop` 的值是 `undefined`，则直接返回 上一次的默认值 `vm._props[key]`，这样可以避免触发不必要的 `watcher` 的更新。

  如果有default值，则返回default值，而**对象或数组的默认值必须从一个工厂函数返回**。

* **断言prop是否有效**`assertProp(prop, key, value, vm, absent)`

  即对配置的高级选项做验证，是否required、是否符合type、是否符合validator自定义函数，不符合将发出警告warn

#### 响应式

响应式`defineReactive(props, key, value)`在initData中再仔细总结，主要作用就是为prop设置getter和setter

但在此之前执行了`toggleObserving(false)`，做响应式优化，因为子组件的prop值始终指向父组件的prop值，只要父组件prop变化就会触发子组件重新渲染，因此可以省略子组件prop的observe过程。

#### 代理

`proxy(vm, '_props', key)`作用是当访问this.name的时候相当于访问this._props.name

