# Diff算法
`Diff`算法的前提是发生在相同层级，对比新旧vnode子节点，简化复杂度，时间复杂度为O(n)。

下面用一张示例图来展示：

![img](https://github.com/xiuyuan66/xiuyuan.github.io/blob/master/assets/vue/diff5.png?raw=true)

图上同颜色线框会进行比较：
- 首先比较最上层`a`节点，由于新旧`vnode`树都是`a`，才会继续对其子节点进行比较
- 然后比较第二层节点，新`vnode`树中是节点`b`和`c`,旧`vnode`树是节点`b`和`g`,则会删除节点`g`，添加节点`c`,由于拥有相同的节点`b`，继续对`b`节点的子节点进行比较
- 最后比较第三层节点，新`vnode`树中只有`d`节点，则删除旧`vnode`树中`e`节点

最后我们可以得知只会在相同层级进行`Diff`,只有在相同层级且相同的情况下才会对其子节点进行比较，多余的相同层级节点只会作删除或添加操作

## 源码分析
当响应式数据发生改变时，set方法会让调用Dep.notify通知所有订阅者Watcher，订阅者就会调用patch给真实的DOM打补丁，更新相应的视图。
直接从更新方法`vm._update`开始分析,它的定义在 `src/core/instance/lifecycle.js` 中：
```javascript
  Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    const vm: Component = this //缓存vue实例
    const prevEl = vm.$el //获取真实dom元素
    const prevVnode = vm._vnode //获取旧VNode
    vm._vnode = vnode //保存新VNode，便于下次更新获取
    if (!prevVnode) {
      // 初始化时并没有旧VNode，第一个参数为真实的node节点，直接执行 vm.__patch__
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // 如果存在，那么对prevVnode和vnode进行diff
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    //...
  }
```
每当响应式数据发生改变时，都会先判断有没有旧`VNode`，没有就传入真实的`dom`节点直接执行`vm.__patch__`，否则传入`prevVnode`和`vnode`进行diff，完成更新工作。

上面`_update`函数中有个`VNode`的参数，在 `Vue.js` 中，`Virtual DOM` 是用 `VNode` 这个 `Class` 去描述，我们看看源码是怎么定义的，它定义在 `src/core/vdom/vnode.js` 中：
```javascript
export default class VNode {
  tag: string | void;
  data: VNodeData | void;
  children: ?Array<VNode>;
  text: string | void;
  elm: Node | void;
  ns: string | void;
  context: Component | void; // rendered in this component's scope
  key: string | number | void;
  componentOptions: VNodeComponentOptions | void;
  componentInstance: Component | void; // component instance
  parent: VNode | void; // component placeholder node

  // strictly internal
  raw: boolean; // contains raw HTML? (server only)
  isStatic: boolean; // hoisted static node
  isRootInsert: boolean; // necessary for enter transition check
  isComment: boolean; // empty comment placeholder?
  isCloned: boolean; // is a cloned node?
  isOnce: boolean; // is a v-once node?
  asyncFactory: Function | void; // async component factory function
  asyncMeta: Object | void;
  isAsyncPlaceholder: boolean;
  ssrContext: Object | void;
  fnContext: Component | void; // real context vm for functional nodes
  fnOptions: ?ComponentOptions; // for SSR caching
  devtoolsMeta: ?Object; // used to store functional render context for devtools
  fnScopeId: ?string; // functional scope id support

  constructor (
    tag?: string,
    data?: VNodeData,
    children?: ?Array<VNode>,
    text?: string,
    elm?: Node,
    context?: Component,
    componentOptions?: VNodeComponentOptions,
    asyncFactory?: Function
  ) {
    this.tag = tag
    this.data = data
    this.children = children
    this.text = text
    this.elm = elm
    this.ns = undefined
    this.context = context
    this.fnContext = undefined
    this.fnOptions = undefined
    this.fnScopeId = undefined
    this.key = data && data.key
    this.componentOptions = componentOptions
    this.componentInstance = undefined
    this.parent = undefined
    this.raw = false
    this.isStatic = false
    this.isRootInsert = true
    this.isComment = false
    this.isCloned = false
    this.isOnce = false
    this.asyncFactory = asyncFactory
    this.asyncMeta = undefined
    this.isAsyncPlaceholder = false
  }

  // DEPRECATED: alias for componentInstance for backwards compat.
  /* istanbul ignore next */
  get child (): Component | void {
    return this.componentInstance
  }
}
```
这里我们只需要了解几个核心的属性就行了，例如：
- `tag` 属性即节点标签属性
- `data` 属性是一个存储节点属性的对象，节点上的`class`，`style`以及绑定的事件
- `children` 属性是`vnode`的子节点集合，每个子结点也是`vnode`结构
- `text` 属性是文本节点
- `elm` 属性为这个`vnode`对应的真实`dom`节点的引用
- `key` 属性是`vnode`的标记，在`diff`过程中对比`key`可以提高`diff`的效率

接着看`vm.__patch__`，`vm.__patch__` 方法定义在 `src/core/vdom/patch.js` 中：
```javascript
  return function patch (oldVnode, vnode, hydrating, removeOnly) {
    //...
    if (isUndef(oldVnode)) {
      // 当oldVnode不存在时，创建新的节点
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue)
    } else {
      const isRealElement = isDef(oldVnode.nodeType) //
      if (!isRealElement && sameVnode(oldVnode, vnode)) {//比较新旧节点
        //新旧节点相同时执行patch比较
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
      } else{
      //...
      // 替换已存在的元素
        const oldElm = oldVnode.elm
        const parentElm = nodeOps.parentNode(oldElm)

        // 创建新的节点
        createElm(
          vnode,
          insertedVnodeQueue,
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        )
      // 删除旧节点
        if (isDef(parentElm)) {
          removeVnodes(parentElm, [oldVnode], 0, 0)
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode)
        }
      }
    }

    ...
  }
```
此时对于`patch`过程有了大概的了解，我们来总结一下吧：
- 如果`oldVnode`不存在时，根据`vnode`直接创建新的`dom`节点
- 如果`oldVnode`存在时，首先判断是不是真实的元素，再调用`sameVnode`比较新旧节点是否相同。
   *   当`oldVnode`和`vnode`基本属性相同时，则调用`patchVnode`进行递归比较
   *   当`oldVnode`和`vnode`基本属性不相同时，则根据`vnode`创建新的真实`dom`,同时根据`oldVnode`删除旧dom节点，
   
上面代码中有看到`sameVnode`和 `patchVnode`两个方法，接下来我们来看看这两个方法有什么软用：
## sameVnode
```javascript
  function sameVnode (a, b) {
    return (
      a.key === b.key && (//唯一标识key相同
        (
          a.tag === b.tag && //标签名相同
          a.isComment === b.isComment &&//是否为注释
          isDef(a.data) === isDef(b.data) &&//数据是否不为空
          sameInputType(a, b)//input类型比较
        ) || (
          isTrue(a.isAsyncPlaceholder) &&
          a.asyncFactory === b.asyncFactory &&
          isUndef(b.asyncFactory.error)//异步组件判断asyncFactory是否相同
        )
      )
    )
  }
```
主要是以下几个属性进行比较：
- 新旧vnode的key是否相同
- 是否同为注释isComment
- data属性是否不为空
- input类型是否相同
- 异步组件的asyncFactory是否相同

## patchVnode
接着重头戏`patchVnode`方法，来看看源码是怎么实现的吧。
```javascript
  function patchVnode (
    oldVnode,
    vnode,
    insertedVnodeQueue,
    ownerArray,
    index,
    removeOnly
  ) {
    if (oldVnode === vnode) { // 当新旧节点完全一样，则直接返回
      return
    }
    // ...
    //保存旧 vnode 的 DOM 引用
    const elm = vnode.elm = oldVnode.elm;
    // ...
    let i;
    const data = vnode.data;
    const oldCh = oldVnode.children; // 获取oldVnode的子节点
    const ch = vnode.children; // 获取vnode的子节点
    // ...
    // 当vnode不是一个文本节点时
    if (isUndef(vnode.text)) { 
      // 当oldVNode和VNode都有子节点时
      if (isDef(oldCh) && isDef(ch)) { 
        // 并且子节点不等时，再递归执行updateChildren比较子节点
        if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly) 
      // 当只有vnode的子节点存在时
      } else if (isDef(ch)) { 
        // ...
        // 当oldVnode为文本节点时，设置旧的 vnode 的内容为空
        if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
        // 根据位置添加插入新的DOM节点
        addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue); 
      // 当只有oldVnode的子节点存在时 
      } else if (isDef(oldCh)) { 
        // 直接移除所有子节点
        removeVnodes(elm, oldCh, 0, oldCh.length - 1);
      // 当只有oldVnode存在并且是文本节点时 
      } else if (isDef(oldVnode.text)) { 
        // 直接置空文本处理
        nodeOps.setTextContent(elm, ''); 
      }
    } else if (oldVnode.text !== vnode.text) { 
      // 当vnode是文本节点，如果文本内容和oldVnode不一样时，直接将oldVnode节点设置为vnode节点文本内容
      nodeOps.setTextContent(elm, vnode.text); 
    }
    
     ...
  }
```
从以上源码得知`diff`过程：
- 首先判断`vnode`是否是文本节点，若`oldVnode.text !== vnode.text`,用`vnode`的文本替换真实`dom`节点的内容
- 当`vnode`没有文本节点时，则开始进行子节点的比较
- 当`vnode`的子节点和`oldVnode`的子节点都存在且不相同的情况下，递归调用`updateChildren`进行更新子节点
- 只有`vnode`的子节点存在，`oldVnode`有文本时，清空dom中的文本，同时调用`addVnodes`方法把`vnode`的子节点添加到真实dom中
- 若只有`oldVnode`的子节点存在时，则直接清空真实`dom`下对应的`oldVnode`子节点
- 若`oldVnode`存在且有文本节点，直接清空对应的文本
  
## updateChildren
上面提到了`updateChildren`方法，这才是`diff`的核心算法，我们一起来看下到底干了什么：
```javascript
  function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
    let oldStartIdx = 0;                  // 旧节点开始位置
    let newStartIdx = 0;                  // 新节点开始位置
    let oldEndIdx = oldCh.length - 1;     // 旧节点结束位置
    let oldStartVnode = oldCh[0];         // 旧节点未处理的第一个节点(旧头)
    let oldEndVnode = oldCh[oldEndIdx];   // 旧节点未处理的最后一个节点(旧尾)
    let newEndIdx = newCh.length - 1;     // 新节点结束位置
    let newStartVnode = newCh[0];         // 新节点未处理的第一个节点(新头)
    let newEndVnode = newCh[newEndIdx];   // 新节点未处理的最后一个节点(新尾)
    
    let oldKeyToIdx, idxInOld, vnodeToMove, refElm
    //...
    //遍历 oldCh 和 newCh 来比较和更新
    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
      if (isUndef(oldStartVnode)) {//第一个节点为空，右移下标
        oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
      } else if (isUndef(oldEndVnode)) {//最后一个节点为空，左移下标
        oldEndVnode = oldCh[--oldEndIdx]
      } else if (sameVnode(oldStartVnode, newStartVnode)) {//同位置比较，如果旧头和新头相同时，继续执行patchVnode递归下去
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        //旧头和新头位置同时右移
        oldStartVnode = oldCh[++oldStartIdx]
        newStartVnode = newCh[++newStartIdx]
      } else if (sameVnode(oldEndVnode, newEndVnode)) {//同位置比较，如果旧尾和新尾相同时，继续执行patchVnode递归下去
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        //旧尾和新尾位置同时左移
        oldEndVnode = oldCh[--oldEndIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldStartVnode, newEndVnode)) { //不同位置比较（需要移动），如果旧头和新尾相同时，先执行patchVnode递归下去，再执行insertBefore插入相应位置的真实dom节点
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
        oldStartVnode = oldCh[++oldStartIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldEndVnode, newStartVnode)) {//不同位置比较（需要移动），如果旧尾和新头相同时，先执行patchVnode递归下去，再执行insertBefore插入相应位置的真实dom节点
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
        oldEndVnode = oldCh[--oldEndIdx]
        newStartVnode = newCh[++newStartIdx]
      } else {//如果新旧头尾都不同时，建立key-->index的对应关系，判断新节点是否在旧节点的集合中，有就移动相应位置，否则直接创建新的节点插入
        if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)// 如果 oldKeyToIdx 不存在，创建 old children 中 vnode 的 key 到 index 的
      // 映射，方便我们之后通过 key 去拿下标。
        idxInOld = isDef(newStartVnode.key)
          ? oldKeyToIdx[newStartVnode.key]
          : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)//判断新节点是否在旧节点中，并获取相应索引下标
        if (isUndef(idxInOld)) { //如果不存在则直接新建一个真实dom节点
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        } else {//存在则先判断两个节点是否相同
          vnodeToMove = oldCh[idxInOld]//根据索引下标获取旧节点
          if (sameVnode(vnodeToMove, newStartVnode)) {//当两个节点相等时，执行patchVnode递归下去，再执行insertBefore插入相应位置的真实dom节点
            patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
            oldCh[idxInOld] = undefined
            canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
          } else {//相同key但是节点不同时，直接创建新的节点
            // same key but different element. treat as new element
            createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
          }
        }
        newStartVnode = newCh[++newStartIdx]
      }
    }
    //上面循环结束后，处理可能未处理到的节点
    if (oldStartIdx > oldEndIdx) {//新节点还有未处理的，遍历剩余的新节点并逐个新增到真实DOM中
      refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
      addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
    } else if (newStartIdx > newEndIdx) {//旧节点还有未处理完的，删除对应的dom
      removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx)
    }
  }
```

具体`diff`过程分析：

保存四个变量`oldStartIdx`、`oldEndIdx`、`newStartIdx`、`newEndIdx`来作为遍历的索引，当`oldCh` 或者 `newCh`的`startIndex > endIndex`，循环结束。

接下来我们来一一分析：

- 旧头新头比较：对新旧两个头部进行比较，如果相同，继续进行`patchVnode`操作，头部索引向右移动
- 旧尾新尾比较：对新旧两个尾部部进行比较，如果相同，继续进行`patchVnode`操作，尾部索引向左移动
- 旧头新尾比较：头尾交叉进行比较，如果相同，继续进行`patchVnode`操作，旧头索引向右移动，新尾索引向左移动
- 旧尾新头比较：头尾交叉进行比较，如果相同，继续进行`patchVnode`操作，旧尾索引向左移动，新头索引向右移动
- 4 个 vnode 都不相同，那么我们就要利用key对比：
    * 从 `oldCh` 数组建立 `key --> index` 的 map映射。
    * 只处理 `newStartVnode` （简化逻辑，有循环我们最终还是会处理到所有 `vnode`），通过它的 `key` 从上面的 `map` 里拿到 `index`；
    * 如果 `index` 不存在，那么说明 `newStartVnode` 是全新的 `vnode`，直接
        创建对应的 `dom` 并插入。
    * 如果 `index` 存在，并且是相同的节点，继续进行`patchVnode`操作，再执行`insertBefore`插入相应位置的真实`dom`节点；
    * 如果 `index` 存在，并且是不同的节点，直接
        创建对应的 `dom` 并插入；

我们再通过示意图来理解以上过程：

![img](https://github.com/xiuyuan66/xiuyuan.github.io/blob/master/assets/vue/diff.png?raw=true)

首先进行旧头新头比较，都是`A`，所以双方头部索引向右移

![img](https://github.com/xiuyuan66/xiuyuan.github.io/blob/master/assets/vue/diff1.png?raw=true)

旧头新头再继续比较，发现不一样，`B!==C`,所以进入旧尾新尾比较，`D===D`,双方尾部索引向左移

![img](https://github.com/xiuyuan66/xiuyuan.github.io/blob/master/assets/vue/diff2.png?raw=true)

现在进入新的循环，旧头新头比较，`B!==C`,旧尾新尾比较，`C!==E`,进入头尾交叉比较，先进行旧尾新头比较`C===C`,旧尾索引左移，新头索引右移

![img](https://github.com/xiuyuan66/xiuyuan.github.io/blob/master/assets/vue/diff3.png?raw=true)

紧接着再进入新一轮的循环，旧头新头比较，`B!==F`,旧尾新尾比较，`B!==E`,头尾交叉比较，`B!==E`,`B!==F`,四种都不相同，这个时候需要通过`key`去比对，然后将新头右移，重复循环直至任一头部索引大于尾部索引，循环结束。

![img](https://github.com/xiuyuan66/xiuyuan.github.io/blob/master/assets/vue/diff4.png?raw=true)

上述循环结束后，可能存在未处理的vnode
- `oldStartIdx > oldEndIdx`,说明`oldCh`先处理完，`newCh`还有未处理完的，添加`newCh`中未处理的节点
- `newStartIdx > newEndIdx`,说明`oldCh`未处理完，删除(`oldCh`中对应的)多余的dom
  
## 参考文献
- [解析vue2.0的diff算法](https://github.com/aooy/blog/issues/2)
- [详解vue的diff算法](https://juejin.im/post/6844903607913938951#heading-3) 
- [深入剖析：Vue核心之虚拟DOM](https://github.com/fengshi123/blog/issues/10) 

