## 节流
```javascript
function throttle(fn, wait) {
  let timer = undefined;
  let lastCallTime = Date.now();
  return function(...args) {
    const timeSinceLastCall = Date.now() - lastCallTime;
    const shouldCall = timeSinceLastCall >= wait;
    if (shouldCall) {
      clearTimeout(timer);
      timer = setTimeout(function() {
        fn.apply(this, args);
      }, wait);
      lastCallTime = Date.now();
    }
  }
}
```
## 防抖
```javascript
function debounce(fn, delay) {
  let timer = null;
  let isNow = true;
  return () => {
    if (isNow) {
      fn.apply(this, arguments);
      isNow = false;
      return;
    }
    if (timer) clearTimeout(timer);
    timer = setTimeout(() => {
      fn.apply(this, arguments);
    }, delay);
  };
}

```
## 模拟new操作
```javascript
  function _new(Func,...args){
    let obj = {}
    // let [constructor,...args] = [...arguments]
    Object.setPrototypeOf(obj, Func.prototype)
    // obj._proto_ = Func.prototype
    let res = Func.apply(obj,args)
    if((res&& typeof res === 'function')||typeof res === 'object'){
      return res
    }
    return obj
  }
  const a = create(Person, '修远', 66)
  console.log(a.name) // '修远'
  console.log(a.age) // 66
  a.sayName() // '修远'

```
## call
```javascript
  Function.prototype.myCall = function(context, ...args) {
    // 执行上下文都保证是对象类型，如果不是就是window
    context = Object(context) || window;
    // 创建一个额外的变量当做context的属性
    const fn = Symbol();
    // 给这个fn属性赋值为当前的函数
    context[fn] = this;
    // 执行函数把...args传入
    const result = context[fn](...args);
    // 删除使用过的fn属性
    delete context[fn];
    // 返回函数执行结果
    return result;
  };

```
## apply
```javascript
  Function.prototype.myApply = function(context, arrArgs) {
    context = Object(context) || window;
    const fn = Symbol();
    context[fn] = this;
    // 需要把传入apply的数组进行展开运算
    // 所以在这里性能会有些消耗相比call来讲
    const result = context[fn](...arrArgs);
    delete context[fn];
    return result;
  }

```
## bind
```javascript
  Function.prototype.myBind = function(context,...args){
    return(...newArgs)=>{
      return this.call(context,...args,...newArgs)
    }
  }

```
## Promise
```javascript
  /*
    Promise构造函数
    executor:执行器函数
  */
  class Promise {
    constructor(exector) {
      // 初始化状态
      this.status = 'pending';
      // 存储成功、失败结果
      this.value = undefined;
      // 成功态回调函数队列
      this.resolvedCallbacks = [];
      // 失败态回调函数队列
      this.rejectedCallbacks = [];

      const resolve = value => {
        // 只有进行中状态才能更改状态
        if (this.status === 'pending') {
          // 将状态改为fulfilled
          this.status = 'fulfilled';
          // 保存value的值
          this.value = value;
          // 如果有待执行的成功态callback函数，立即异步执行回调函数
          if (this.rejectedCallbacks.length>0){
              setTimeout(()=>{
                this.resolvedCallbacks.forEach(fn => fn(this.value));
              })
          }
          
        }
      }
      const reject = value => {
        // 只有进行中状态才能更改状态
        if (this.status === 'pending') {
          // 将状态改为rejected
          this.status = 'rejected';
          // 保存value的值
          this.value = value;
          // 如果有待执行的失败态callback函数，立即异步执行回调函数
          if (this.rejectedCallbacks.length>0){
              setTimeout(()=>{
                this.rejectedCallbacks.forEach(fn => fn(this.value))
              })
          }
          
        }
      }
      try {
        // 立即执行executor
        exector(resolve, reject);
      } catch(e) {
        // 如果执行器抛出异常，promise对象变为rejected状态
        reject(e);
      }
    }
    /*
    指定一个成功/失败的回调函数
    返回一个新的promise对象
    */
    then(onFulfilled, onRejected) {
      onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : value => value;
      onRejected = typeof onRejected === 'function'? onRejected: reason => { throw reason }
      // 保存this
      const self = this;
      return new Promise((resolve, reject) => {
        /*
        调用指定回调函数的处理，根据执行结果。改变return的promise状态
          */
        function handle(callback,fn) {
          // try捕获错误
            try{
              // 模拟微任务
              setTimeout(() => {
                const result = callback(self.value)
                // 1. 如果回调函数返回的不是promise，调用新Promise的resolve函数。
                // 2. 如果回调函数返回的是promise，return的promise的结果就是这个promise的结果
                result instanceof Promise ? result.then(resolve, reject) : fn(result);
              })
            }catch (e) {
                //  3.如果执行onResolved的时候抛出错误，则返回的promise的状态为rejected
                reject(e)
            }
        }
        if (self.status === 'pending') {
          self.resolvedCallbacks.push(() => {
            handle(onFulfilled,resolve)
          });
          self.rejectedCallbacks.push(() => {
            handle(onRejected,reject)
          })
        } else if (self.status === 'fulfilled') {
          handle(onRejected,resolve)
        } else{// 当status === 'rejected'
          handle(onRejected,reject)
        }
      });
    }
    /*
      Promise原型对象的.catch
      指定一个失败的回调函数
      返回一个新的promise对象
      */
    catch(onRejected) {
      return this.then(undefined, onRejected);
    }
    /*
      Promise函数对象的resovle方法
      返回一个指定结果的promise对象
      */
    static resolve(value) {
      if (value instanceof Promise) {
        // 如果value是Promise实例，直接返回
        return value;
      } else {
        // 如果不是Promise实例，返回一个新的Promise对象，状态为fulfilled
        return new Promise((resolve, reject) => resolve(value));
      }
    }
    /*
      Promise函数对象的reject方法
      返回一个指定reason的失败状态的promise对象
      */
    static reject(reason) {
      return new Promise((resolve, reject) => {
        reject(reason);
      })
    }
    /*
    Promise的all方法
    返回一个promise对象，只有当所有promise都成功时返回的promise状态才成功
    */
    static all = function(promises){
      const len = promises.length
      const values = new Array(len)
      var resolvedCount = 0 //记录状态为fulfilled的promise的数量
      return new Promise((resolve,reject)=>{
          // 遍历promises，获取每个promise的结果
          promises.forEach((p,index)=>{
            //强制转换为promise实例，确保每一个都是promise实例
              Promise.resolve(p).then(
                  val => {
                      // p状态为resolved，将值保存起来
                      values[index] = val
                      resolvedCount++;
                      // 如果全部p都为fulfilled状态，return的promise状态为fulfilled
                      if(resolvedCount === len){
                          resolve(values)
                      }
                  },
                  err => { //只要有一个失败，return的promise状态就为rejected
                      reject(err)
                  }
              )
          })
      })
    }

    /*
    Promise的race方法
    返回一个promise对象，状态由第一个完成的promise决定
    */
    static race = function(promises){
      return new Promise((resolve,reject)=>{
          // 遍历promises，获取每个promise的结果
          promises.forEach((p,index)=>{
              Promise.resolve(p).then(
                  val => {
                      // 只要有一个成功，返回的promise的状态就是fulfilled
                      resolve(val)
                  },
                  err => { //只要有一个失败，return的promise状态就为rejected
                      reject(err)
                  }
              )
          })
      })
    }
  }


```
## 深拷贝
```javascript
  function cloneDeep(source, hash = new WeakMap()) {
    //对传入参数进行校验
    if (!isObject(source)) return source;
    //哈希表存在直接返回
    if (hash.has(source)) return hash.get(source);
    //支持函数
    if (source instanceof Function) {
      return function () {
        source.apply(this, arguments);
      };
    }
    // 支持日期
    if (source instanceof Date) return new Date(source);
    // 支持正则对象
    if (source instanceof RegExp) return new RegExp(source.source, source.flags);

    let target = Array.isArray(source) ? [] : {};
    //哈希表设值
    hash.set(source, target);
    //获取所有的键值，同时包括 Symbol，对 source 遍历赋值即可
    Reflect.ownKeys(source).forEach((key) => {
      if (isObject(source[key])) {
        target[key] = cloneDeep4(source[key], hash);
      } else {
        target[key] = source[key];
      }
    });
    return target;
  }
  function isObject(obj) {
    return typeof obj === "object" && obj != null;
  }

```
## 柯里化函数
指的是将一个接受多个参数的函数 变为 接受一个参数返回一个函数的固定形式，这样便于再次调用，例如f(1)(2)
```javascript
  function curry(fn, args) {
    let len = fn.length;
    let _args = args || [];
    return function(){
        let newArgs = _args.concat(Array.prototype.slice.call(arguments));
        if (newArgs.length < len) {
            return curry.call(this,fn,newArgs);
        }else{
            return fn.apply(this,newArgs);
        }
    }
  }
```
## 继承
```javascript
  // 寄生组合继承
  function inherit(child, parent) {
    let proto = Object.create(parent.prototype); // 父类原型副本
    proto.constructor = child;   // constructor指向子类
    child.prototype = proto;     // 赋给子类的prototype原型
  }
  // 父类
  function Parent(name) {
    this.name = name;
    this.arr = ['八度空间', '叶惠美'];
  }
  Parent.prototype.sayName = function() {
    console.log(this.name);
  };

  // 子类
  function Child(name, age) {
    // 组合还是Parent.call(this, ...args)构造函数继承实例上的属性和方法
    Parent.call(this, name);
    this.age = age;
  }
  // 继承
  inherit(Child, Parent);

  Child.prototype.sayAge = function() {
    console.log(this.age);
  }

  // 测试用例
  let child  = new Child('Jay', 18);
  child.arr.push('范特西');
  child.sayName();
  child.sayAge();
  console.log(child.arr);

  let super1 = new Parent('xuan');
  console.log(super1.arr);
```
## 数组扁平化
```javascript
  //方案1 
  const flatDeep = (arr, d = 1) => {
    //d表示维度
    return d > 0 ? arr.reduce((acc, val) => acc.concat(Array.isArray(val) ? flatDeep(val, d - 1) : val),
    []) :
        arr.slice();
  };

  //方案2 
  const res = [];
  const flatFn = arr => {
    for (let i = 0; i < arr.length; i++) {
      if (Array.isArray(arr[i])) {
        flatFn(arr[i]);
      } else {
        res.push(arr[i]);
      }
    }
  }

```
## 数组去重
```javascript
  //方案1 
  const unique_1 = arr => [...new Set(arr)];
  //方案2 
  const unique_2 = arr => arr.reduce((pre, cur) => pre.includes(cur) ? pre : [...pre, cur], []);
  //方案3 
  const unique_3 = arr=> {
    return res = arr.filter( (item, index, array)=> {
      return arr.indexOf(item) === index;
    })
  }
  //方案4 
  const unique_4 = arr=> {
    var obj = {};
    return array.filter(function (item, index, array) {
        return obj.hasOwnProperty(typeof item + item) ? false : (obj[typeof item + item] = true)
    })
  }

```
## Array.prototype.splice
```javascript
  Array.prototype.mySplice = function (start, deleteCount, ...addList) {
    //start为负时
    if (start < 0) {
      //当start为负且start的绝对值大于length时，则表示开始位置为0，start赋值为0
      if (Math.abs(start) > this.length) {
        start = 0;
      } else {
        start += this.length;
      }
    }
    //当deleteCount未传入时，那么start之后数组的所有元素都会被删除。
    if (typeof deleteCount === "undefined") {
      deleteCount = this.length - start;
    }
    //利用slice方法获取待移除的元素组成的数组
    const removeList = this.slice(start, start + deleteCount);
    //利用slice获取右侧剩余元素
    const right = this.slice(start + deleteCount);

    let addIndex = start;
    // 将待添加元素与右侧剩余元素添加至原数组中
    addList.concat(right).forEach((item) => {
      this[addIndex] = item;
      addIndex++;
    });
    this.length = addIndex;

    return removeList;
  };
```
## instanceof
```javascript
  function myInstanceOf(leftValue,rightValue){
    if( typeof leftValue !== 'object' || leftValue === null) return false;
    let righProto = rightValue.prototype
    let leftProto = leftValue.__proto__
    while(true){
      if(leftProto === null) return false
      if(leftProto === righProto) return true
      leftProto = leftProto.__proto__
    }
  }

```
## 事件总线|发布订阅
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
```
## 图片懒加载
```javascript
  function isVisible(el) {
    
    const bound = el.getBoundingClientRect()//获取元素的大小及位置
    //可视区的高度
    const windowHeight = document.documentElement.clientHeight
    // 顶部边缘可见
    const top = bound.top > 0 && bound.top <= windowHeight;
    // 底部边缘可见
    const bottom = bound.bottom <= windowHeight && bound.bottom > 0;
    return top || bottom;
  }

  function loadImg() {
    const imgs = document.querySelectorAll('img')
    for (let img of imgs) {
      const source = img.dataset.src
      if (!source) continue
      if (isVisible(img)) {
        img.src = source
        img.dataset.src = ''
      }
    }
  }


```
欢迎大家来补充~
