# 发布-订阅模式
## 什么是发布-订阅模式
  实现了对象间的一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都将得到通知，并自动更新

  订阅者（`Subscriber`）把自己想订阅的事件注册（`Subscribe`）到调度中心（`Topic`），当发布者（`Publisher`）发布该事件（`Publish topic`）到调度中心，也就是该事件触发时，由调度中心统一调度（`Fire Event`）订阅者注册到调度中心的处理代码。

  举一个例子，你在社交平台上关注了B，同时其他很多人也关注了B，那么当B发布动态的时候，平台就会为你们推送这条动态。B就是发布者，你是订阅者，平台就是调度中心，你和B是没有直接的消息往来的，全是通过平台来协调的（你的关注，B的发布动态）。
## 如何实现发布-订阅模式
- 创建一个对象作为发布者
- 对象用来缓存队列（调度中心），存放回调函数以便通知订阅者
- 订阅者可以往事件列表添加事件，表示订阅
- 发布消息的时候，发布者遍历事件列表，触发里面的回调函数
- on 方法用来把函数 cb 都加到缓存列表中（订阅者注册事件到调度中心）
- emit 方法取到 type 当做 key，判断 events[type] 去执行对应缓存列表中的函数（发布者发布事件到调度中心，调度中心处理代码）
- off 方法可以根据 event 值取消订阅（取消订阅）
- once 方法只监听一次，调用完毕后删除缓存函数（订阅一次）
### 下面实现一个简易版的发布-订阅
```javascript
  //事件中心对象
  const events = {}
  // 订阅
  const on = (event, fn) => {
      // 如果events[event]中没有对应的事件名为 key 的数组，也就是说明没有订阅过，就给 event 创建个缓存列表,并且把其对应的回调函数放到此数组中
      // 如有events[event]中有相应的 event 值，把 fn 添加到对应 event 的缓存列表里
      (events[event] || (events[event] = [])).push(fn);
  }
  // 发布消息
  const emit = (event, data) => {
    // 遍历 event 值对应的缓存列表，依次执行 fn并传入data参数
      events[event].forEach(item => item(data));
  }

  // 测试
  on('event1', () => { console.log('this is on event1')})
  on('event2', () => { console.log('this is on event2')})
  
  emit('event1') // this is on event1
  emit('event2') // this is on event2


```
### 接下来实现一个全局版的发布-订阅

```javascript
// 发布订阅中心, on-订阅, off取消订阅, emit发布,once监听一次，内部需要一个单独事件中心events进行存储;
class EventEmitter{
  constructor(){
    this.events = Object.create(null)
  }
  // 订阅
  on(event,cb){
    let events = this.events
    // 如果events[event]中没有对应的事件名为 key 的数组，也就是说明没有订阅过，就给 event 创建个缓存列表,并且把其对应的回调函数放到此数组中
      // 如有events[event]中有相应的 event 值，把 fn 添加到对应 event 的缓存列表里
    (events[event] || (events[event] = [])).push(fn);
  }
  // 发布
  emit(event,...args){
    if(this.events[event]){
      // 遍历 event 值对应的缓存列表，依次执行 fn
      this.events[event].forEach(listener => {
        listener.call(this,...args)
      });
    }
  }
   // 取消订阅
  off(event,cb){
    let events = this.events
    if(events[event]){
      //利用filter返回不相同的函数重新赋值给events[event]达到取消订阅的目的
      events[event] = events[event].filter(listener=>{
        return listener !==cb && listener.listen !==cb
      })
    }
  }
  // 只执行一次函数
  once(event,cb){
    //先执行，调用后删除
    function wrap(){
      typeof cb === 'function' && cb.apply(this, args);;
      this.off(event,wrap)
    }
    wrap.listen = cb
    this.on(event,wrap)
  }
}
//测试
const eb = new EventEmitter();
const offFn = () => {
    console.log("这是一个可以被移除的事件");
};
const onceFn = () => {
    console.log("这是一个只触发一次的事件");
};
eb.on("offFn", offFn);
eb.emit("offFn");//这是一个可以被移除的事件
eb.off("offFn", toBeOffFn);
eb.emit("offFn");//已移除，不会触发

eb.once("onceFn", onceFn);//这是一个只触发一次的事件
eb.emit("onceFn");//这是一个只触发一次的事件
eb.emit("onceFn");//不会再执行
```
## Vue中的EventBus
`EventBus` 又称为事件总线。在`Vue`中可以使用 `EventBus` 来作为沟通桥梁的概念，就像是所有组件共用相同的事件中心，可以向该中心注册发送事件或接收事件，所以组件都可以上下平行地通知其他组件，但也就是太方便所以若使用不慎，就会造成难以维护的灾难，因此才需要更完善的`Vuex`作为状态管理中心，将通知的概念上升到共享状态层次。

相信大家在`Vue`项目中遇到过父子组件通信，父组件会通过 props 向下传递给子组件，子组件通过 $emit事件告诉给父组件，我们直接看下源码当中是怎么实现的。

源码位置`src/core/instance/events.js`
```javascript
export function eventsMixin (Vue: Class<Component>) {
  const hookRE = /^hook:/
  // event 参数可以是一个字符串，也可以是一个字符串数组，fn 事件监听的回调
  Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {
    const vm: Component = this
    // 如果是一个字符串数组，则遍历递归数组中的每一项
    if (Array.isArray(event)) {
      for (let i = 0, l = event.length; i < l; i++) {
        vm.$on(event[i], fn)
      }
    } else {
      // 如果 vm._events[event] 不存在则创建对应的事件名为 key 的数组，并且把其对应的回调函数放到此数组中，也就是说一个事件名，会对应一个数组，里面存在了此事件名所对应的所有回调  
      (vm._events[event] || (vm._events[event] = [])).push(fn)
      // optimize hook:event cost by using a boolean flag marked at registration
      // instead of a hash lookup
      if (hookRE.test(event)) {
        // 标记为钩子函数
        vm._hasHookEvent = true
      }
    }
    return vm
  }
// 只执行一次函数
  Vue.prototype.$once = function (event: string, fn: Function): Component {
    const vm: Component = this
    // 定义一个事件回调函数
    function on () {
      // 取消 event 事件的监听
      vm.$off(event, on)
      // 执行传入的函数 fn
      fn.apply(vm, arguments)
    }
    // 给 on 挂载 fn 属性
    on.fn = fn
    // 监听 event 事件，从这里可以知道，$once() 方法其内部实现最终也是通过 vm.$on() 方法实现的，但它是怎么做到多次触发，但只会执行一次的呢？
    // 关键在于上面定义的 on() 方法，这个方法对传进来的事件回调重新封装了一层，而里面的实现则是先取消实例上此事件的监听，而后再执行传入的函数。
    vm.$on(event, on)
    return vm
  }
  // 注销事件函数
  Vue.prototype.$off = function (event?: string | Array<string>, fn?: Function): Component {
    const vm: Component = this
    // 如果没有参数，则注销所有事件
    if (!arguments.length) {
      vm._events = Object.create(null)
      return vm
    }
    // 如果第一个参数为数组，则注销指定 事件
    if (Array.isArray(event)) {
      // 遍历调用（递归） vm.$off()
      for (let i = 0, l = event.length; i < l; i++) {
        vm.$off(event[i], fn)
      }
      return vm
    }
    // 取出已经声明的事件对应的回调函数数组
    const cbs = vm._events[event]
    // 如果事件名不存在
    if (!cbs) {
      return vm
    }
    // 如果要取消的事件名所对应的回调没传，则是取消此事件的所有监听回调
    if (!fn) {
      vm._events[event] = null
      return vm
    }
    // specific handler
    let cb
    let i = cbs.length
    // 遍历找出数组中对应的回调并从数组中删除掉，这样在每次事件被触发时，就不会触发你取消了的这个事件监听回调了，因为它已经不在回调函数数组中
    while (i--) {
      cb = cbs[i]
      if (cb === fn || cb.fn === fn) {
        cbs.splice(i, 1)
        break
      }
    }
    return vm
  }

  Vue.prototype.$emit = function (event: string): Component {
    const vm: Component = this
    if (process.env.NODE_ENV !== 'production') {
      // 转为小写字符串
      const lowerCaseEvent = event.toLowerCase()
      // 如果转换存在大小写，并且父组件有监听此事件的小写事件名，则会打印出提示
      if (lowerCaseEvent !== event && vm._events[lowerCaseEvent]) {
        tip(
          `Event "${lowerCaseEvent}" is emitted in component ` +
          `${formatComponentName(vm)} but the handler is registered for "${event}". ` +
          `Note that HTML attributes are case-insensitive and you cannot use ` +
          `v-on to listen to camelCase events when using in-DOM templates. ` +
          `You should probably use "${hyphenate(event)}" instead of "${event}".`
        )
      }
    }
    // vm._events 是在 Vue.prototype.$on 方法中定义的，
    // 取出对应事件中的回调函数数组
    let cbs = vm._events[event]
    if (cbs) {
      cbs = cbs.length > 1 ? toArray(cbs) : cbs
       // 取出参数。this.$emit() 方法的第一个参数为事件名，后面的都被视为参数，所以在使用 this.$emit() 时，我们可以传任意个参数
      const args = toArray(arguments, 1)
      const info = `event handler for "${event}"`
      // 遍历执行数组中的每一个函数
      for (let i = 0, l = cbs.length; i < l; i++) {
        // invokeWithErrorHandling 只是对函数进行了封装
        invokeWithErrorHandling(cbs[i], vm, args, vm, info)
      }
    }
    return vm
  }
}

```

经典的事件中心的实现，把所有的事件保存在`vm._events`，当执行 `vm.$on(event,fn)` 的时候，通过事件的名称 `event` 把回调函数 `fn` 存储在对应的数组 `vm._events[event].push(fn)`。当执行 `vm.$emit(event)` 的时候，根据事件名 `event` 判断是否存在并找到所有的回调函数 `let cbs = vm._events[event]`，然后遍历执行所有的回调函数。当执行 `vm.$off(event,fn)` 的时候会遍历移除指定事件名 `event` 和指定的 `fn `。当执行 `vm.$once(event,fn)` 的时候，内部就是执行 `vm.$on`，并且当回调函数执行一次后再通过 `vm.$off` 移除事件的回调，这样就确保了回调函数只执行一次。

## 总结
### 优点
- 时间上的解耦: 在异步编程中，由于无法确定异步加载的时间，有可能订阅事件的模块还没有初始化完毕而异步加载就完成了，发布者就已经发布事件了。通过发布订阅模式，可以将发布者的事件提前保存起来，等到发布者加载完毕再执行。
- 对象间的解耦：发布订阅模式中，发布者和订阅者可以不必知道对方的存在，而是通过中介对象来通信。
### 缺点
- 创建订阅者本身需要一定的时间和内存，而当你订阅一个消息后，也许此消息最后都未发生，但这个订阅者会始终存在于内存中。
- 另外，发布订阅模式将对象间完全解耦，如果过度使用的话，对象和对象之间的必要联系就会被掩盖，会导致程序难以追踪和理解。

## 拓展
### 浅谈观察者模式与发布-订阅模式
![img](https://github.com/xiuyuan66/Blog/blob/master/assets/event/pattern.png?raw=true) 
### 观察者模式
观察者（`Observer`）直接订阅（`Subscribe`）主题（`Subject`），而当主题被激活的时候，会触发（`Fire Event`）观察者里的事件。
### 发布-订阅模式
订阅者（`Subscriber`）把自己想订阅的事件注册（`Subscribe`）到调度中心（`Topic`），当发布者（`Publisher`）发布该事件（`Publish topic`）到调度中心，也就是该事件触发时，由调度中心统一调度（`Fire Event`）订阅者注册到调度中心的处理代码。
### 两者区别
可以看出，发布订阅模式相比观察者模式多了个事件通道，事件通道作为调度中心，管理事件的订阅和发布工作，彻底隔绝了订阅者和发布者的依赖关系。即订阅者在订阅事件的时候，只关注事件本身，而不关心谁会发布这个事件；发布者在发布事件的时候，只关注事件本身，而不关心谁订阅了这个事件。

观察者模式有两个重要的角色，即目标和观察者。在目标和观察者之间是没有事件通道的。一方面，观察者要想订阅目标事件，由于没有事件通道，因此必须将自己添加到目标(`Subject`) 中进行管理；另一方面，目标在触发事件的时候，也无法将通知操作(`notify`) 委托给事件通道，因此只能亲自去通知所有的观察者。

如果有不足之处或者观点不一致的地方，欢迎指出和讨论 ~
