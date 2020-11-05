## 前言

​每当被问到 Vue 数据双向绑定原理的时候，大家可能都会脱口而出：Vue 内部通过 `Object.defineProperty`方法属性拦截的方式，把 `data` 对象里每个数据的读写转化成 `getter`/`setter`，当数据变化时通知视图更新。虽然一句话把大概原理概括了，但是其内部的实现方式还是值得深究的，本文分析 `Vue` 源码的数据双向绑定实现原理，来进一步巩固加深对数据双向绑定的理解认识。

## MVVM 数据双向绑定

所谓MVVM数据双向绑定，即主要是：数据变化更新视图，视图变化更新数据。如下图：

![img](https://github.com/xiuyuan66/xiuyuan.github.io/blob/master/assets/vue/mvvm/view.jpg?raw=true)

- 输入框内容变化时，`Data` 中的数据同步变化。即 `View` => `Data` 的变化。
- `Data` 中的数据变化时，文本节点的内容同步变化。即 `Data` => `View` 的变化。

要实现这两个过程，关键点在于数据变化如何更新视图，因为视图变化更新数据我们可以通过事件监听的方式来实现。所以我们着重讨论数据变化如何更新视图。

接下来直接看`Vue`源码是怎么实现的吧。

## Vue 源码
**initState**

在 `Vue` 的初始化阶段，`_init` 方法执行的时候，会执行 `initState(vm)` 方法，它的定义在 `src/core/instance/state.js` 中。 

```javascript
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)//method代理到vm实例上
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  //注意初始化顺序，data/computed中可以访问props和method
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

`initState` 方法主要是对 `props`、`methods`、`data`、`computed` 和 `wathcer` 等属性做了初始化操作。这里我们主要分析 `data`，首先会进行判断`opts.data` 是否有`data`这个属性，该`data`就是我们的在`Vue`实例化的时候传进来的，如果有就会调用`initData(vm)`方法，没有就调用`observe(vm._data = {}, true /* asRootData */)`方法，我们先来看看`initData(vm)`方法。

**initData**

```javascript
function initData (vm: Component) {
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    )
  }
  // proxy data on instance
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) {
      proxy(vm, `_data`, key)//令 vm.key === vm.props.key
    }
  }
  // 监听整个data
  // data数据改动最为频繁，而且data可能会有很深的{}层次
  observe(data, true /* asRootData */)
}
```

`data` 的初始化主要过程是做两件事，一个是对定义 `data` 函数返回对象的遍历，通过 `proxy` 把每一个值 `vm._data.xxx` 都代理到 `vm.xxx` 上，所以我们就可以通过 `vm.xxxx` 访问到定义在 `data` 函数返回对象中的 `xxxx` 属性了；另一个是调用 `observe` 方法观测整个 `data` 的变化，把 `data` 也变成响应式，我们接下去主要介绍 `observe` 。 

**observe**

`observe` 的功能就是用来监测数据的变化，它的定义在 `src/core/observer/index.js` 中： 

```javascript
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```
`observe` 方法的作用就是给`data`创建一个`Observer`实例并返回，这里面涉及了一些判断条件：
首先会判断`data`是不是对象类型或者是不是`VNode`类型，如果不是对象类型或者是`VNode`类型直接`return`,接着判断`data`是否有`__ob__`属性，如果有`__ob__`属性的话，说明已经实例化过`Observer`，`ob = value.__ob__`赋值完直接返回该实例，否则`ob = new Observer(value)`,实例化`Observer`，`data`都会有一个`__ob__`属性，里面存放了属性的`Observer`实例，目的就是防止重复绑定。接下来我们来看一下 `Observer` 的作用。 
 

**Observer**

`Observer` 是一个类，它的作用是给对象的属性添加 getter 和 setter，用于依赖收集和派发更新： 

```javascript
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

  /**
   * Walk through all properties and convert them into
   * getter/setters. This method should only be called when
   * value type is Object.
   */
  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

  /**
   * Observe a list of Array items.
   */
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```
代码往下执行，我们就会执行 `def(value, '__ob__', this)` 这句代码，把自身实例添加到数据对象 `value` 的 `__ob__ `属性上，`def` 的定义在 `src/core/util/lang.js` 中：
```javascript
export function def (obj: Object, key: string, val: any, enumerable?: boolean) {
  Object.defineProperty(obj, key, {
    value: val,
    enumerable: !!enumerable,
    writable: true,
    configurable: true
  })
}
```
如上代码我们可以看得到，`def` 函数是一个非常简单的`Object.defineProperty` 的封装，我们使用了 `Object.defineProperty`这样的方法监听对象`obj`中的 `__ob__` 这个`key`。但是`obj`对象中又没有该`key`，因此`Object.defineProperty`会在该对象上定义一个新属性为 `__ob__`, 也就是说，如果我们的数据被 `Object.defineProperty`绑定过的话，那么绑定完成后，就会有 `__ob__`这个属性，因此我们之前通过了这个属性来判断是否已经被绑定过了。

再回到`Observer` ，可以看到`Observer` 的构造函数逻辑比较简单，首先实例化 `Dep` 对象， `Dep` 对象，之后会介绍。接下来会对 `value` 做数组判断，如果不是数组，就执行`walk` 方法，如果是，就判断`hasProto`是否为`true`（也就是判断浏览器是否支持_`_proto__`属性），如果为`true`，则调用`protoAugment`方法，否则调用`copyAugment`，对于数组会直接调用 `observeArray` 方法。可以看到 `observeArray` 是遍历数组再次调用 `observe` 方法，而 `walk` 方法是遍历对象的 `key` 调用 `defineReactive` 方法，那么我们首先来看一下前面提到的`protoAugment(value, arrayMethods)`这个方法是做什么的。

**protoAugment**

```javascript
function protoAugment (target, src: Object) {
  /* eslint-disable no-proto */
  target.__proto__ = src
  /* eslint-enable no-proto */
}
```
把目标对象的`__proto__`指向传进来的参数`arrayMethods`,它的定义在 `src/core/observer/array.js` 中：
**arrayMethods**
```javascript
import { def } from '../util/index'

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

/**
 * Intercept mutating methods and emit events
 */
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
    // 通知改变
    ob.dep.notify()
    return result
  })
})
```
结合之前的`protoAugment(value, arrayMethods)`方法，目的是让 `value`对象参数指向了 `arrayMethods` 原型上的方法，然后我们使用 `Obejct.defineProperty`去监听数组中的原型方法，当我们在`data`对象参数调用数组方法，比如`push，unshift`等方法就可以理解为映射到 `arrayMethods` 原型上，因此会被 `Object.defineProperty`方法监听到。因此会执行对应的`set/get`方法。

如上 `if (inserted) ob.observeArray(inserted)` 代码中，为什么针对 方法为 `push, unshift, splice` 等一些数组新增的元素也会调用 `ob.observeArray(inserted)` 进行响应性变化，`inserted` 参数为一个数组。也就是说我们不仅仅对`data`现有的元素进行响应性监听，还会对数组中一些新增删除的元素也会进行响应性监听。...`args`运算符会转化为数组。

如果 `hasProto`为`false`， 也就是说浏览器不支持`__proto__`这个属性的话，我们就会调用`copyAugment(value, arrayMethods, arrayKeys)` 方法去处理，源码如下：

```javascript
const arrayKeys = Object.getOwnPropertyNames(arrayMethods)//["push", "shift", "unshift", "pop", "splice", "reverse", "sort"]
function copyAugment (target: Object, src: Object, keys: Array<string>) {
  for (let i = 0, l = keys.length; i < l; i++) {
    const key = keys[i]
    def(target, key, src[key])
  }
}
```
如上可以看到，对数组的方法进行遍历，然后调用`def`方法进行监听，对于`def`方法前面已经解释过了，这里不再赘述，回到`Observer`实例中，如果不是数组，调用`walk` 方法，而`walk` 方法是遍历对象的 `key` 调用 `defineReactive` 方法，接下来我们看下`defineReactive`方法。

**defineReactive**

`defineReactive` 的功能就是定义一个响应式对象，给对象动态添加 `getter` 和 `setter`，它的定义在 `src/core/observer/index.js` 中： 

```javascript
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()
  // 获取属性自身的描述符
  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  /*
   检查属性之前是否设置了 getter / setter
   如果设置了，则在之后的 get/set 方法中执行 设置了的 getter/setter
  */
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }

  let childOb = !shallow && observe(val)//递归循环该val, 判断是否还有子对象，如果还有子对象的话，就继续实列化该value，
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      // 如果属性原本拥有getter方法的话则执行该方法
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          // 如果有子对象的话，对子对象进行依赖收集
          childOb.dep.depend()
          // 如果value是数组的话，则递归调用
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      /*
       如果属性原本拥有getter方法则执行。然后获取该值与newValue对比，如果相等的
       话，直接return，否则的值，执行赋值。
      */
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
        // 如果属性原本拥有setter方法的话则执行
        setter.call(obj, newVal)
      } else {
        // 如果属性原本没有setter方法则直接赋新值
        val = newVal
      }
      // 继续判断newVal是否还有子对象，如果有子对象的话，继续递归循环遍历
      childOb = !shallow && observe(newVal)
      // 有值发生改变的话，我们需要通知所有的订阅者
      dep.notify()
    }
  })
}
```

`defineReactive` 函数最开始初始化 `Dep` 对象的实例，接着拿到 `obj` 的属性描述符，然后对子对象递归调用 `observe` 方法，这样就保证了无论 `obj` 的结构多复杂，它的所有子属性也能变成响应式的对象，这样我们访问或修改 `obj` 中一个嵌套较深的属性，也能触发 getter 和 setter，最后利用`Object.defineProperty`给`key`添加`getter` 和 `setter`,`getter`做的事情是依赖收集，`setter` 做的事情是派发更新，。 

### 6.2、订阅器 Dep 实现

订阅器`Dep` 是整个 `getter` 依赖收集的核心，它的定义在 `src/core/observer/dep.js` 中： 

```javascript
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  depend () {
    if (Dep.target) {
      // Watcher也收集Dep
      Dep.target.addDep(this)
    }
  }

  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}

// The current target watcher being evaluated.
// This is globally unique because only one watcher
// can be evaluated at a time.

// Dep.target设置，同一时间只能有一个全局的 `Watcher` 被计算
Dep.target = null
const targetStack = []

export function pushTarget (target: ?Watcher) {
  targetStack.push(target)
  Dep.target = target
}

export function popTarget () {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}
```

`Dep` 是一个 `Class`，它定义了一些属性和方法，这里需要特别注意的是它有一个静态属性 `target`，这是一个全局唯一 `Watcher`，这是一个非常巧妙的设计，因为在同一时间只能有一个全局的 `Watcher` 被计算，另外它的自身属性 `subs` 也是 `Watcher` 的数组。`Dep` 实际上就是对 `Watcher` 的一种管理，`Dep` 脱离 `Watcher` 单独存在是没有意义的。

### 6.3、订阅者 Watcher 实现

订阅者`Watcher` 的一些相关实现，它的定义在 `src/core/observer/watcher.js` 中 

```javascript
export default class Watcher {
  vm: Component;
  expression: string;
  cb: Function;
  id: number;
  deep: boolean;
  user: boolean;
  lazy: boolean;
  sync: boolean;
  dirty: boolean;
  active: boolean;
  deps: Array<Dep>;
  newDeps: Array<Dep>;
  depIds: SimpleSet;
  newDepIds: SimpleSet;
  before: ?Function;
  getter: Function;
  value: any;

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
      if (!this.getter) {
        this.getter = noop
        process.env.NODE_ENV !== 'production' && warn(
          `Failed watching path: "${expOrFn}" ` +
          'Watcher only accepts simple dot-delimited paths. ' +
          'For full control, use a function instead.',
          vm
        )
      }
    }
    this.value = this.lazy
      ? undefined
      : this.get()
  }

  /**
   * Evaluate the getter, and re-collect dependencies.
   */
  get () {
    // 设置Dep.target
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

  /**
   * Add a dependency to this directive.
   */
  addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        // dep增加Watcher
        dep.addSub(this)
      }
    }
  }

  /**
   * Clean up for dependency collection.
   */
  cleanupDeps () {
    let i = this.deps.length
    while (i--) {
      const dep = this.deps[i]
      if (!this.newDepIds.has(dep.id)) {
        dep.removeSub(this)
      }
    }
    let tmp = this.depIds
    this.depIds = this.newDepIds
    this.newDepIds = tmp
    this.newDepIds.clear()
    tmp = this.deps
    this.deps = this.newDeps
    this.newDeps = tmp
    this.newDeps.length = 0
  }

  /**
   * Subscriber interface.
   * Will be called when a dependency changes.
   */
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

  /**
   * Scheduler job interface.
   * Will be called by the scheduler.
   */
  run () {
    if (this.active) {
      const value = this.get()
      if (
        value !== this.value ||
        // Deep watchers and watchers on Object/Arrays should fire even
        // when the value is the same, because the value may
        // have mutated.
        isObject(value) ||
        this.deep
      ) {
        // set new value
        const oldValue = this.value
        this.value = value
        if (this.user) {
          try {
            this.cb.call(this.vm, value, oldValue)
          } catch (e) {
            handleError(e, this.vm, `callback for watcher "${this.expression}"`)
          }
        } else {
          this.cb.call(this.vm, value, oldValue)
        }
      }
    }
  }

  /**
   * Evaluate the value of the watcher.
   * This only gets called for lazy watchers.
   */
  evaluate () {
    this.value = this.get()
    this.dirty = false
  }

  /**
   * Depend on all deps collected by this watcher.
   */
  depend () {
    let i = this.deps.length
    while (i--) {
      this.deps[i].depend()
    }
  }

  /**
   * Remove self from all dependencies' subscriber list.
   */
  teardown () {
    if (this.active) {
      // remove self from vm's watcher list
      // this is a somewhat expensive operation so we skip it
      // if the vm is being destroyed.
      if (!this.vm._isBeingDestroyed) {
        remove(this.vm._watchers, this)
      }
      let i = this.deps.length
      while (i--) {
        this.deps[i].removeSub(this)
      }
      this.active = false
    }
  }
}
```

`Watcher` 是一个 `Class`，在它的构造函数中，定义了一些和 `Dep` 相关的属性 ，其中，`this.deps` 和 `this.newDeps` 表示 `Watcher` 实例持有的 `Dep` 实例的数组；而 `this.depIds` 和 `this.newDepIds` 分别代表 `this.deps` 和 `this.newDeps` 的 `id` `Set`数据结构 。

**过程分析**

当对数据对象的访问会触发他们的 `getter` 方法，那么这些对象什么时候被访问呢？ `Vue` 的 `mount` 过程是通过 `mountComponent` 函数，其中有一段比较重要的逻辑，大致如下：

```javascript
updateComponent = () => {
  vm._update(vm._render(), hydrating)
}
new Watcher(vm, updateComponent, noop, {
  before () {
    if (vm._isMounted) {
      callHook(vm, 'beforeUpdate')
    }
  }
}, true /* isRenderWatcher */)
```

当我们去实例化一个渲染 `watcher` 的时候，首先进入 `watcher` 的构造函数逻辑，然后会执行它的 `this.get()` 方法，进入 `get` 函数，首先会执行： 

```javascript
pushTarget(this)
```

实际上就是把 `Dep.target` 赋值为当前的渲染 `watcher` 并压栈（为了恢复用）。接着又执行了：

```javascript
value = this.getter.call(vm, vm)
```
`this.getter` 对应就是 `updateComponent` 函数，这实际上就是在执行：

```javascript
vm._update(vm._render(), hydrating)
```
它会先执行 `vm._render()` 方法，这个方法会生成渲染 `VNode`(这里先当作是虚拟`dom`)，并且在这个过程中会对 `vm` 上的数据访问，这个时候就触发了数据对象的 `getter`。

然而每个对象值的 `getter` 都持有一个 `dep`，在触发 `getter` 的时候会调用 `dep.depend()` 方法，也就会执行 `Dep.target.addDep(this)`。

刚才我们提到这个时候 `Dep.target` 已经被赋值为渲染 `watcher`，那么就执行到 `addDep` 方法：

```javascript
addDep (dep: Dep) {
  const id = dep.id
  if (!this.newDepIds.has(id)) {
    this.newDepIds.add(id)
    this.newDeps.push(dep)
    if (!this.depIds.has(id)) {
      // dep增加Watcher
      dep.addSub(this)
    }
  }
}
```

这时候会做一些逻辑判断（保证同一数据不会被添加多次）后执行 `dep.addSub(this)`，那么就会执行 `this.subs.push(sub)`，也就是说把当前的 `watcher` 订阅到这个数据持有的 `dep` 的 `subs` 中，这个目的是为后续数据变化时候能通知到哪些 `subs` 做准备。所以在 `vm._render()` 过程中，会触发所有数据的 `getter`，这样实际上已经完成了一个依赖收集的过程。在完成依赖收集后，还有几个逻辑要执行，首先是：

```javascript
if (this.deep) {
  traverse(value)
}
```
这个是要递归去访问 `value`，触发它所有子项的 `getter`。接下来执行：

```javascript
popTarget()
```
`popTarget` 的定义在 `src/core/observer/dep.js` 中：
```javascript
export function popTarget () {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}
```
实际上就是把 `Dep.target` 恢复成上一个状态，因为当前 `vm` 的数据依赖收集已经完成，那么对应的渲染`Dep.target` 也需要改变。最后执行：
```javascript
this.cleanupDeps()
```
这里主要是清空依赖，下面详细分析一下源码过程：
```javascript
cleanupDeps () {
    let i = this.deps.length
    while (i--) {
      const dep = this.deps[i]
      if (!this.newDepIds.has(dep.id)) {
        dep.removeSub(this)
      }
    }
    let tmp = this.depIds
    this.depIds = this.newDepIds
    this.newDepIds = tmp
    this.newDepIds.clear()
    tmp = this.deps
    this.deps = this.newDeps
    this.newDeps = tmp
    this.newDeps.length = 0
  }
```
`Watcher` 在构造函数中会初始化 2 个 `Dep` 实例数组，`newDeps` 表示新添加的 `Dep` 实例数组，而 `deps` 表示上一次添加的 `Dep` 实例数组。
在执行 `cleanupDeps` 函数的时候，会首先遍历 `deps`，移除对 `dep.subs` 数组中 `Wathcer` 的订阅，然后把 `newDepIds` 和 `depIds` 交换，`newDeps` 和 `deps` 交换，并把 `newDepIds` 和 `newDeps` 清空。`Vue` 设计了在每次添加完新的订阅，会移除掉旧的订阅。

当我们在组件中对响应的数据做了修改，就会触发 `setter` 的逻辑，最后调用 `watcher` 中的 `update` 方法： 

```javascript
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

在一般组件数据更新的场景，会走到最后一个 `queueWatcher(this)` 的逻辑，`queueWatcher` 的定义在 `src/core/observer/scheduler.js` 中：

```javascript
export const MAX_UPDATE_COUNT = 100

const queue: Array<Watcher> = []
const activatedChildren: Array<Component> = []
let has: { [key: number]: ?true } = {}
let circular: { [key: number]: number } = {}
let waiting = false
let flushing = false
let index = 0

export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // queue the flush
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      nextTick(flushSchedulerQueue)
    }
  }
}
```
这里引入了一个队列的概念，这也是 `Vue` 在做派发更新的时候的一个优化的点，它并不会每次数据改变都触发 `watcher` 的回调，而是把这些 `watcher` 先添加到一个队列里，然后在 `nextTick` 后执行 `flushSchedulerQueue`。

这里有几个细节要注意一下，首先用 `has` 对象保证同一个 `Watcher` 只添加一次；接着对 `flushing` 的判断，`else` 部分的逻辑稍后我会讲；最后通过 `waiting` 保证对 `nextTick(flushSchedulerQueue)` 的调用逻辑只有一次，另外 `nextTick` 的实现之后另写一篇文章专门去讲，目前就可以理解它是在下一个 `tick`，也就是异步的去执行 `flushSchedulerQueue`。

接下来我们来看 `flushSchedulerQueue` 的实现，它的定义在 `src/core/observer/scheduler.js` 中。

```javascript
function flushSchedulerQueue () {
  flushing = true
  let watcher, id

  // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child)
  // 2. A component's user watchers are run before its render watcher (because
  //    user watchers are created before the render watcher)
  // 3. If a component is destroyed during a parent component's watcher run,
  //    its watchers can be skipped.
  queue.sort((a, b) => a.id - b.id)

  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before()
    }
    id = watcher.id
    has[id] = null
    watcher.run()
    // in dev build, check and stop circular updates.
    if (process.env.NODE_ENV !== 'production' && has[id] != null) {
      circular[id] = (circular[id] || 0) + 1
      if (circular[id] > MAX_UPDATE_COUNT) {
        warn(
          'You may have an infinite update loop ' + (
            watcher.user
              ? `in watcher with expression "${watcher.expression}"`
              : `in a component render function.`
          ),
          watcher.vm
        )
        break
      }
    }
  }

  // keep copies of post queues before resetting state
  const activatedQueue = activatedChildren.slice()
  const updatedQueue = queue.slice()

  resetSchedulerState()

  // call component updated and activated hooks
  callActivatedHooks(activatedQueue)
  callUpdatedHooks(updatedQueue)

  // devtool hook
  /* istanbul ignore if */
  if (devtools && config.devtools) {
    devtools.emit('flush')
  }
}
```
这里有几个重要的逻辑要梳理一下，对于一些分支逻辑如 keep-alive 组件相关和之前提到过的 updated 钩子函数的执行会略过。

**队列排序**

`queue.sort((a, b) => a.id - b.id)` 对队列做了从小到大的排序，这么做主要有以下要确保以下几点：

1.组件的更新由父到子；因为父组件的创建过程是先于子的，所以 `watcher` 的创建也是先父后子，执行顺序也应该保持先父后子。

2.用户的自定义 `watcher` 要优先于渲染 `watcher` 执行；因为用户自定义 `watcher` 是在渲染 `watcher` 之前创建的。

3.如果一个组件在父组件的 `watcher` 执行期间被销毁，那么它对应的 `watcher` 执行都可以被跳过，所以父组件的 `watcher` 应该先执行。

**队列遍历**

在对 `queue`排序后，接着就是要对它做遍历，拿到对应的 `watcher`，执行 `watcher.run()`。这里需要注意一个细节，在遍历的时候每次都会对 `queue.length` 求值，因为在 `watcher.run()` 的时候，很可能用户会再次添加新的 `watcher`，这样会再次执行到 `queueWatcher`，如下：

```javascript
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // ...
  }
}
```
可以看到，这时候 `flushing`为 `true`，就会执行到 `else` 的逻辑，然后就会从后往前找，找到第一个待插入 `watcher` 的 `id` 比当前队列中 `watcher` 的 `id `大的位置。把 `watcher` 按照 `id`的插入到队列中，因此 `queue` 的长度发生了变化。

这个过程就是执行 `resetSchedulerState` 函数，它的定义在 `src/core/observer/scheduler.js` 中。

```javascript
const queue: Array<Watcher> = []
const activatedChildren: Array<Component> = []
let has: { [key: number]: ?true } = {}
let circular: { [key: number]: number } = {}
let waiting = false
let flushing = false
let index = 0

/**
 * Reset the scheduler's state.
 */
function resetSchedulerState () {
  index = queue.length = activatedChildren.length = 0
  has = {}
  if (process.env.NODE_ENV !== 'production') {
    circular = {}
  }
  waiting = flushing = false
}
```
逻辑非常简单，就是把这些控制流程状态的一些变量恢复到初始值，把 `watcher` 队列清空。

接下来我们继续分析 `watcher.run()` 的逻辑，它的定义在 `src/core/observer/watcher.js` 中。

```javascript
class Watcher{
  /**
   * Scheduler job interface.
   * Will be called by the scheduler.
   */
  run () {
    if (this.active) {
      const value = this.get()
      if (
        value !== this.value ||
        // Deep watchers and watchers on Object/Arrays should fire even
        // when the value is the same, because the value may
        // have mutated.
        isObject(value) ||
        this.deep
      ) {
        // set new value
        const oldValue = this.value
        this.value = value
        if (this.user) {
          try {
            this.cb.call(this.vm, value, oldValue)
          } catch (e) {
            handleError(e, this.vm, `callback for watcher "${this.expression}"`)
          }
        } else {
          this.cb.call(this.vm, value, oldValue)
        }
      }
    }
  }
}
```
`run` 函数逻辑也很简单，先通过 `this.get()` 得到它当前的值，然后做判断，如果满足新旧值不等、新值是对象类型、`deep` 模式任何一个条件，则执行 `watcher` 的回调，注意回调函数执行的时候会把第一个和第二个参数传入新值 `value `和旧值 `oldValue`，这就是当我们添加自定义 `watcher` 的时候能在回调函数的参数中拿到新旧值的原因。

那么对于渲染 `watcher` 而言，它在执行 `this.get()` 方法求值的时候，会执行 `getter` 方法：

```javascript
updateComponent = () => {
  vm._update(vm._render(), hydrating)
}
```

所以这就是当我们去修改组件相关的响应式数据的时候，会触发组件重新渲染的原因，接着就会重新执行 `patch` 的过程，但它和首次渲染有所不同。

### Vue 数据双向绑定原理图

以上主要分析了 Vue 数据双向绑定的关键代码，其原理图可以表示如下：

![4.png](https://github.com/xiuyuan66/xiuyuan.github.io/blob/master/assets/vue/mvvm/4.png?raw=true)  



## 总结

data通过Observer转换成了getter/setter的形式来追踪变化，当外界通过Watcher读取数据时，会触发getter从而将Watcher添加到依赖中。当数据发生了变化时，会触发setter，从而向Dep中的依赖（Watcher）发送通知。Watcher接收到通知后，会向外界发送通知，变化通知到外界后可能会触发视图更新，也有可能触发用户的某个回调函数等。


## 参考文献

- [0 到 1 掌握：Vue 核心之数据双向绑定](https://juejin.im/post/6844903903822086151)

- [Vue 技术揭秘](https://ustbhuangyi.github.io/vue-analysis/)

 

 

 