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
    if (oldVnode === vnode) { // 当新旧节点相同时，则直接返回
      return
    }
    // ...
    //保存旧 vnode 的 DOM 引用
    const elm = vnode.elm = oldVnode.elm;
    // ...
    let i;
    const data = vnode.data;
    const oldCh = oldVnode.children; // 获取oldVnode的子节点集合
    const ch = vnode.children; // 获取vnode的子节点集合
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
    if (isDef(data)) {
      // 根据新vnode更新旧vnode的选项配置、数据属性、propsData等
      if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode); 
    }
    // ...
  }
```
