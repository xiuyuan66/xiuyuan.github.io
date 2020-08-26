## Hooks 初识

官方提供的钩子

目前官方提供的钩子共分为两种，分为基本钩子以及拓展钩子

基本钩子共有：`useState` 、`useEffect` 、 `useContext`

额外的钩子有：`useCallback` 、 `useReducer` 、  `useMemo` 、 `useRef` 、  `useLayoutEffect` 、 `useImperativeHandle` 、  `useDebugValue`


## useState
```javascript
  import React, { useState } from 'react';

  const Component = () => {
    const [count, setCount] = useState(0);
    
    return (
      <button onClick={() => setCount(count + 1)}>click<button>
    )
  }

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
```javascript
  useEffect(() => {
    console.log(count);
  }, [count]);

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