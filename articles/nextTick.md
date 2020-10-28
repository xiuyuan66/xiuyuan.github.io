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

当然 MutationObserver 也是有浏览器兼容的，我们可以使用如下代码来检测浏览器是否支持该属性，如下代码:

```javascript
let MutationObserver = window.MutationObserver || window.WebkitMutationObserver || window.MozMutationObserver;
// 监测浏览器是否支持
let observeMutationSupport = !!MutationObserver;
```
首先我们要使用 MutationObserver 构造函数的话，我们先要实例化 MutationObserver 构造函数，同时我们要指定该实列的回调函数，如下代码：

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