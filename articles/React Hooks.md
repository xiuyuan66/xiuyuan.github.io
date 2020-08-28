

## Hooks的定义
Hook 是什么呢？下面是官方给出的定义：
```javascript
Hooks are functions that let you “hook into” React state and lifecycle features from function components
```

Hook 本质上是函数，它能够让函数组件也拥有状态和生命周期的特性。这意味着我们原本只能使用 class 组件实现的多数功能，现在用函数组件也能实现。不过 Hook 的使用范围也有一定限制，限定于函数组件和自定义 hook 中。

Hooks 名称统一约定以 use 前缀开头（比如 usexxx），因为 React 还无法自动识别哪些是普通 JavaScript 函数，哪些是 Hooks 函数。下面我们将介绍函数组件和 React Hooks 是如何解决 class 组件的痛点的。
## Hooks 初识

官方提供的钩子

目前官方提供的钩子共分为两种，分为基本钩子以及拓展钩子

基本钩子共有：`useState` 、`useEffect` 、 `useContext`

额外的钩子有：`useCallback` 、 `useReducer` 、  `useMemo` 、 `useRef` 、  `useLayoutEffect` 、 `useImperativeHandle` 、  `useDebugValue`
## useState
该钩子用于给函数组件创建一个新的状态，useState参数为一个固定的值或者一个有返回值的函数作为状态的初始值。钩子执行后的结果为一个数组，分别为生成的状态以及改变该状态的方法，通过解构赋值的方法拿到对应的值与方法，以下示例可以看出使用函数组件再也不用关心 super(props) 和 this 指针问题。
```javascript
  import React, { useState } from 'react';

  const Component = () => {
    const [count, setCount] = useState(0);
    
    return (
      <button onClick={() => setCount(count + 1)}>click<button>
    )
  }  
```
- `count` 为状态的初始值， 相当于 `this.state`
- `setCount`为改变`count`的方法，相当于`this.setState`参数可以是一个新的`state`值，也可以是有返回值的方法
- `setCount`也可以命名为其他名字，执行的时候直接替换原来的`state`，而不是原来的合并
- `setCount` 可能是异步的过程，所以如果更新`state`依赖之前的`state`，请使用函数作为`setCount`的参数来更新`state`
- `useState` 是在函数组件或者自定义`Hook`中使用的，所以无论在`mount`还是`update`的时候都会被执行，但是`update`过程的执行不会重新生成`state`，而是返回最新的`state`和`setState`方法；
```javascript
  let _state;
  function useState(initialState) {
    _state = _state || initialState; // 如果存在旧值则返回， 使得多次渲染后的依然能保持状态。
    function setState(newState) {
      _state = newState;
      render();  // 重新渲染，将会重新执行 Counter
    }
    return [_state, setState];
  }

```
```javascript
  function Counter() {
    const [count, setCount] = useState(0);
    return (
      <div>
        <div>{count}</div>
        <button onClick={() => setCount(count + 1)}>
          点击
        </button>
      </div>
    );
  }

```
## useEffect
顾名思义，执行副作用钩子。主要用于以下两种情况：

- 函数式组件中不存在传统类组件生命周期的概念，如果我们需要在一些特定的生命周期或者值变化后做一些操作的话，必须借助  useEffect  的一些特性去实现。

- useState  产生的 changeState 方法并没有提供类似于  setState  的第二个参数一样的功能，因此如果需要在 State 改变后执行一些方法，必须通过  useEffect  实现。

该钩子接受两个参数，第一个参数为副作用需要执行的回调，生成的回调方法可以返回一个函数（将在组件卸载时运行）；第二个为该副作用监听的状态数组，当对应状态发生变动时会执行副作用，如果第二个参数为空，那么在每一个 State 变化时都会执行该副作用。

```javascript
import React, { useState, useEffect } from "react";
import { Button, message } from "antd";
const Component = () => {
  const [count, setCount] = useState(0);
  useEffect(() => {
    message.info(`组件挂载,最新值${count}`);
  }, [count]);
  return (
    <div>
      <div>{count}</div>
      <Button onClick={() => setCount(count + 1)}>click</Button>
    </div>
  )
};
```
![img](https://github.com/workerxuan/workerxuan.github.io/blob/master/assets/react/useEffect1.gif?raw=true)
```javascript
  import React, { useState, useEffect } from 'react';
  import { Button, message } from "antd";
  let timer = null;
  const Component = ({ visible }) => {
    useEffect(() => {
      timer = setInterval(() => {
        // events ...
        message.info(`组件挂载`);
      }, 1000);

      return () => {
        // 类似 componentWillUnmount
        // unmount events ...
        message.info(`组件卸载`);
        clearInterval(timer); // 组件卸载 移除计时器
      };
    }, []);
    return visible ? 'true' : 'false';
  };
  const ParentDemo = () => {
    const [visible, changeVisible] = useState(true);
    return (
      <div>
        {
          visible && <Component visible={visible} />
        }
        <button onClick={() => { changeVisible(!visible); }}>
          改变visible
        </button>
      </div>
    );

  }
  // ...
}

```
```javascript
import React, { useState, useEffect } from "react";
import { Button, message } from "antd";
const Component = () => {
  const [count, setCount] = useState(0);
  useEffect(() => {
    message.info(`组件挂载,最新值${count}`);
  }, [count]);
  return (
    <div>
      <div>{count}</div>
      <Button onClick={() => setCount(count + 1)}>click</Button>
    </div>
  )
};
  function Counter() {
    const [count, setCount] = useState(0);
    useEffect(() => {
    console.log(count);
    }, [count]);
    return (
    <div>
      <div>{count}</div>
      <button onClick={() => setCount(count + 1)}>
        点击
      </button>
    </div>
    );
  }

```
## useMemo
```javascript
  function Counter() {
    const [count, setCount] = useState(0);
    const computed = () => {
      console.log('我执行了');
      return count * 10 - 2;
    }
    const sum = useMemo(computed, [count]);
    return (
      <div>
        <div>{count} * 10 - 2 = {sum}</div>
        <button onClick={() => setCount(count + 1)}>
          点击
        </button>
      </div>
    );
  }


```
```javascript
  let _deps = {
    args: []
  }; // _deps 记录 useMemo 上一次的 依赖
  function useMemo(callback, args) {
    const hasChangedDeps = args.some((arg, index) => arg !== _deps.args[index]); // 两次的 dependencies 是否完全相等
    // 如果 dependencies 不存在，或者 dependencies 有变化
    if (!_deps.args || hasChangedDeps) {
      _deps.args = args;
      _deps._callback = callback;
      _deps.value = callback();
      return _deps.value;
    }

    return _deps.value;
  }


```
## useCallback
```javascript
  function useCallback(callback, args) {
    return useMemo(() => callback, args);
  }
```
```javascript
  function Counter() {
    const [count, setCount] = useState(0);
    useEffect(() => {
    console.log(count);
    }, [count]);
    return (
    <div>
      <div>{count}</div>
      <button onClick={() => setCount(count + 1)}>
        点击
      </button>
    </div>
    );
  }

```