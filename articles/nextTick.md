## 浅谈 nextTick
`nextTick`接受一个回调函数作为参数，它的作用是将回调延迟到下次`dom`更新周期之后执行。如果没有提供回调且在支持Promise的环境中，则返回一个`Promise`。
我们在开发项目的时候会遇到一些需要用到`nextTick`的场景：当更新了状态数据之后，需要对新`dom`做一些操作，但是这时获取不到更新后的`dom`，因为还没有重新渲染。这个时候需要`nextTick`方法。

示例如下：

```javascript
new Vue({
  //...
  methods: {
    //...
    example(){
      //修改数据
      this.msg = '我是测试'
      this.$nextTick(()=>{
        //dom现在更新了
        this.doSomethingElse()
      })
    }
  } 
})
 
```
在`Vue`中，当状态发生变化时，`watcher`会得到通知，然后触发虚拟`dom`的渲染流程。而`watcher`触发渲染这个操作并不是同步的，而是异步的。`Vue`中有一个队列，每当需要渲染时，会将`watcher`推送到这个队列中，在下一次事件循环中再让`watcher`触发渲染的流程。

**为什么使用异步更新队列**

`Vue 2.0`开始使用虚拟`dom`进行渲染，变化侦测的通知只发送到组件，组件内用到的所有状态的变化都会通知到同一个`watcher`，然后虚拟`dom`会将整个组件进行`diff`并更改`dom`。也就是说，如果在同一事件循环中有两个数据发生了变化，那么组件的`watcher`会收到两份通知，从而进行两次渲染。事实上，并不需要渲染两次，虚拟`dom`会将整个组件进行渲染，所以只需要等所有状态都修改完毕后，一次性将整个组件的`dom`渲染到最新即可。

要解决这个问题，`Vue`的实现方式是将收到通知的`watcher`实例添加到队列中缓存起来，并且在添加队列之前检查其中是否已经存在相同的`watcher`，只有不存在时，才将`watcher`实例添加到队列中。然后在下一次事件循环中（`event loop`）中，`Vue`会让队列中的`watcher`触发渲染流程并清空队列。这样就可以保证即便在同一事件循环中有两个状态发生改变，`watcher`最后也只执行一次渲染流程。

**什么是事件循环**

`JavaScript`是一门单线程且非阻塞的脚本语言，这意味着`JavaScript`代码在执行的任何时候都只有一个主线程来处理所有任务。而非阻塞是指当代码需要处理异步任务时，主线程会挂起(`pending`)这个任务，当异步任务处理完毕后，主线程再根据一定规则去执行相应回调。

事实上，当任务处理完毕后，`JavaScript`会将这个事件加入一个队列，我们称之为事件队列。被放入事件队列中的事件不会立即执行其回调，而是等待当前执行栈中的所有任务执行完毕后，主线程会去查找事件队列中是否有任务。

异步任务有两种类型：微任务（`microtask`）和 宏任务（`macrotask`）。不同类型的任务会被分配到不同任务队列中。

当执行栈中的所有任务都执行完毕后，会去检查微任务队列中是否有事件发生，如果存在，则会依次执行微任务队列中事件对应的回调，直到为空。然后去宏任务队列中取出一个事件，把对应的回调加入当前执行栈，当执行栈中的所有任务都执行完毕后，检查微任务队列中是否有任务存在。无限重复此过程，就形成一个无限循环，这个循环就叫作事件循环。

下图就是主线程和事件队列的示意图:

![img](https://github.com/xiuyuan66/xiuyuan.github.io/blob/master/assets/vue/nextTick/eventloop.jpg?raw=true)

回到前面说的下次`dom`更新，其实是下次微任务执行时更新`dom`。而`vm.$nextTick`其实是将回调函数添加到微任务中。只有在特殊情况下才会降级成宏任务，默认会添加到微任务中。

`macrotasks(宏任务)`: 
- setTimeout
- setInterval
- setImmediate
- I/O
- UI rendering
- ...

`microtasks(微任务)`:
- Promise
- process.nextTick
- MutationObserver
- ...
  
**什么是执行栈**

当我们执行一个方法时，`JavaScript`会生成一个与这个方法对应的执行环境（`context`）。又叫执行下文。这个执行环境中有这个方法的的私有作用域、上层作用域的指向、方法的参数、私有作用域中定义的变量以及`this`对象。这个执行环境会被添加到一个栈中，这个栈就是执行栈。

如果在这个方法的代码中执行到了一行函数调用语句，那么`JavaScript`会生成这个函数的执行环境并将其添加到执行栈中，然后进入这个执行环境继续执行其中的代码。执行完毕并返回结果后，`JavaScript`会退出执行环境并把这个执行环境从栈中销毁，回到上一个方法的执行环境。这个过程反复进行，知道执行栈中的代码全部执行完毕。这个执行环境的栈就是执行栈。


### 源码实现
源码如下：
```javascript
/* @flow */
/* globals MutationObserver */

import { noop } from 'shared/util'
import { handleError } from './error'
import { isIE, isIOS, isNative } from './env'

const callbacks = []
let pending = false

function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}

// Here we have async deferring wrappers using microtasks.
// In 2.5 we used (macro) tasks (in combination with microtasks).
// However, it has subtle problems when state is changed right before repaint
// (e.g. #6813, out-in transitions).
// Also, using (macro) tasks in event handler would cause some weird behaviors
// that cannot be circumvented (e.g. #7109, #7153, #7546, #7834, #8109).
// So we now use microtasks everywhere, again.
// A major drawback of this tradeoff is that there are some scenarios
// where microtasks have too high a priority and fire in between supposedly
// sequential events (e.g. #4521, #6690, which have workarounds)
// or even between bubbling of the same event (#6566).
let timerFunc

// The nextTick behavior leverages the microtask queue, which can be accessed
// via either native Promise.then or MutationObserver.
// MutationObserver has wider support, however it is seriously bugged in
// UIWebView in iOS >= 9.3.3 when triggered in touch event handlers. It
// completely stops working after triggering a few times... so, if native
// Promise is available, we will use it:
/* istanbul ignore next, $flow-disable-line */
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  timerFunc = () => {
    p.then(flushCallbacks)
    // In problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    if (isIOS) setTimeout(noop)
  }
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  // PhantomJS and iOS 7.x
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
  // Use MutationObserver where native Promise is not available,
  // e.g. PhantomJS, iOS7, Android 4.4
  // (#6466 MutationObserver is unreliable in IE11)
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, {
    characterData: true
  })
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  // Fallback to setImmediate.
  // Techinically it leverages the (macro) task queue,
  // but it is still a better choice than setTimeout.
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {
  // Fallback to setTimeout.
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}

export function nextTick (cb?: Function, ctx?: Object) {
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

如上代码，我们从上往下看，首先定义变量 `callbacks = []`; 该变量的作用是: 用来存储所有需要执行的回调函数。`let pending = false`; 该变量的作用是表示状态，判断是否有正在执行的回调函数。
也可以理解为，如果代码中 `timerFunc` 函数被推送到任务队列中去则不需要重复推送。

`flushCallbacks()` 函数，该函数的作用是用来执行`callbacks`里面存储的所有回调函数。如下代码:

```javascript
function flushCallbacks () {
  /*
   设置 pending 为 false, 说明该 函数已经被推入到任务队列或主线程中。需要等待当前
   栈执行完毕后再执行。
  */
  pending = false;
  // 拷贝一个callbacks函数数组的副本
  const copies = callbacks.slice(0)
  // 把函数数组清空
  callbacks.length = 0
  // 循环该函数数组，依次执行。
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}
```
这里使用 `callbacks` 而不是直接在 `nextTick` 中执行回调函数的原因是保证在同一个 `tick` 内多次执行 `nextTick`，不会开启多个异步任务，而把这些异步任务都压成一个同步任务，在下一个 `tick` 执行完毕。

接下来代码做了四个判断：对当前环境进行不断的降级处理，尝试使用原生的`Promise.then`、`MutationObserver`和`setImmediate`，上述三个都不支持最后使用`setTimeout`；降级处理的目的都是将`flushCallbacks`函数放入微任务或者宏任务，等待下一次事件循环时来执行。



**MutationObserver**

`MutationObserver` 中文含义可以理解为 "变动观察器"。它是监听`DOM`变动的接口，`DOM`发生任何变动，`MutationObserver`会得到通知。在`Vue`中是通过该属性来监听`DOM`更新完毕的。

它和事件类似，但有所不同，事件是同步的，当`DOM`发生变动时，事件会立刻处理，但是 `MutationObserver` 则是异步的，它不会立即处理，而是等页面上所有的`DOM`完成后，会执行一次，如果页面上要操作100次`DOM`的话，如果是事件的话会监听100次`DOM`，但是我们的 `MutationObserver` 只会执行一次，它是等待所有的`DOM`操作完成后，再执行。

**它的特点是：**

-  等待所有脚本任务完成后，才会执行，即采用异步方式。
-  DOM的变动记录会封装成一个数组进行处理。
-  还可以观测发生在DOM的所有类型变动，也可以观测某一类变动。

当然 `MutationObserver` 也是有浏览器兼容的，我们可以使用如下代码来检测浏览器是否支持该属性，如下代码:

```javascript
let MutationObserver = window.MutationObserver || window.WebkitMutationObserver || window.MozMutationObserver;
// 监测浏览器是否支持
let observeMutationSupport = !!MutationObserver;
```
首先我们要使用 `MutationObserver` 构造函数的话，我们先要实例化 `MutationObserver` 构造函数，同时我们要指定该实列的回调函数，如下代码：

```javascript
let observer = new MutationObserver(callback);
```

观察器`callback`回调函数会在每次`DOM`发生变动后调用，它接收2个参数，第一个是变动的数组，第二个是观察器的实例。

**MutationObserver 实例的方法**

`observe()` 该方法是要观察`DOM`节点的变动的。该方法接收2个参数，第一个参数是要观察的`DOM`元素，第二个是要观察的变动类型。

调用方式为：`observer.observe(dom, options)`;

`options` 类型有如下：

`childList`: 子节点的变动。
`attributes`: 属性的变动。
`characterData`: 节点内容或节点文本的变动。
`subtree`: 所有后代节点的变动。

需要观察哪一种变动类型，需要在`options`对象中指定为`true`即可; 但是如果设置`subtree`的变动，必须同时指定`childList`, `attributes`, 和 `characterData` 中的一种或多种。

### 实现一个简易的nextTick

```javascript
// 存储回调函数
let callbacks = [];
// 代表等待状态的标志位
let pending = false;

function nextTick (cb) {
    callbacks.push(cb);

    if (!pending) {
        pending = true;
        setTimeout(flushCallbacks, 0);
    }
}

function flushCallbacks () {
    pending = false;
    const copies = callbacks.slice(0);
    callbacks.length = 0;
    for (let i = 0; i < copies.length; i++) {
        copies[i]();
    }
}

```
根据源码实现一个简易版，先定义一个`callbacks`存储`nextTick`的回调函数，`pending`是一个标记位，代表等待的状态，然后在`nextTick`里面利用`setTimeout`创建异步任务，`setTimeout` 会在 `task` 中创建一个事件 `flushCallbacks` ，`flushCallbacks` 则会在执行时将 `callbacks` 中的所有 `cb` 依次执行。

## 参考文献

- [Vue系列---理解Vue.nextTick使用及源码分析(五)](https://www.cnblogs.com/tugenhua0707/p/11756584.html#_labe4)

- [Vue.js异步更新及nextTick](https://juejin.im/post/6844903666420318216)