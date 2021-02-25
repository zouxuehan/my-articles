### Set

* set类似于数组，但是成员值都是唯一的
* 可用于数组去重，内部判断是否重复**类似全等操作，但是NaN===NaN**
* WeakSet的成员只能是对象，是弱引用，不计入垃圾回收机制，可能随时消失

### Map

* Map类似于对象，是键值对的集合，但是键值可以为任意数据结构

### Proxy

在目标对象之前架设一层“拦截”，外界对对象的访问都需要经过它，代理某些操作。这样使得JS的使用自由度更高，可以更大限度满足开发需求。

Proxy的应用：跟踪属性、数据绑定、观察对象等

### Reflect

* 将Object的方法都放在了Reflect中，并对返回结果进行修改使其更合理
* Reflect方法与Proxy方法一一对应，就让Proxy对象可以方便地调用对应的Reflect方法，完成默认行为，作为修改行为的基础。

**Proxy和Reflect实现观察者模式**

```js
const queuedObservers = new Set()
const observe = fn => queuedObservers.add(fn)
const observable = obj => new Proxy(obj,{
    set(target,key,value){
        const result = Reflect.set(...arguments)
        queuedObservers.forEach(observe => observe())
        return result
    }
})
const person = observable({
  name: '张三',
  age: 20
});

function print() {
  console.log(`${person.name}, ${person.age}`)
}

observe(print);
person.name = '李四';
```

使用observable创建应该被观察对象person，observe创建观察者print，当person改变时，执行观察者函数

### Promise

es5中异步编程的方式主要是回调函数和事件，存在问题如果一个接口依赖另一个接口返回，那只能回调嵌套，非常深的回调嵌套就成了回调地狱；

es6为了解决回调地狱而实现了Promise

Promise是一个容器，里面保存着某个未来才会结束的事件的结果。

### iterator遍历器

iterator是一个对象（一次性接口），该对象与可迭代对象相关联，iterator有一个私有属性表示可迭代对象中的成员位置，next()方法返回当前位置成员，并使位置+1

```js
function makeIterator(array) {
    var nextIndex = 0;
    return {
        next: function() {
            return nextIndex < array.length ?
            {value: array[nextIndex++], done: false} :
            {value: undefined, done: true};
        }
    };
}
var it = makeIterator(['a', 'b']);

it.next() // { value: "a", done: false }
it.next() // { value: "b", done: false }
it.next() // { value: undefined, done: true }

```

es6新增了set、map这些新的数据结构，但是不能使用for执行遍历，因此为所有可遍历的对象都部署了Iterator接口，部署在[symbol.iterator]属性上

### Generator

Generator 函数是一个状态机，封装了多个内部状态。执行 Generator 函数会返回一个遍历器对象，可以依次遍历 Generator 函数内部的每一个状态，但是**只有调用next方法才会遍历下一个内部状态**，所以其实提供了一种可以暂停执行的函数。

理解：Generator 函数执行产生的上下文环境，一旦遇到yield命令，就会暂时退出堆栈，但是并不消失，里面的所有变量和对象会冻结在当前状态。等到对它执行next命令时，这个上下文环境又会重新加入调用栈，冻结的变量和对象恢复执行。

### async

async函数可以理解为内置自动执行器的Generator函数语法糖，

### module

es6之前，js没有模块体系，主要使用CommonJS(用于服务器)和AMD(用于浏览器)。es6实现模块功能，取代了它们。

区别：

* CommonJS和AMD都只能在运行时加载（动态加载），而es6模块引入为静态加载（编译阶段加载）
* CommonJS模块输出的是值的拷贝，es6模块输出的是值的引用

### es6扩展

1. let、const声明变量
2. 变量解构赋值
3. 新增模板字符串，使拼接更直观
4. 数组新增扩展运算符
5. 函数引入rest参数
6. 新增箭头函数