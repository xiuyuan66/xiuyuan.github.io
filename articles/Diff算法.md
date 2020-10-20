组件更新调用`vm._update`,它的定义在 `src/core/instance/lifecycle.js` 中：
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
每次组件更新时，都会先判断有没有旧`VNode`，没有就传入真实的`dom`节点直接执行`vm.__patch__`，否则传入`prevVnode`和`vnode`进行diff，完成更新工作。

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
