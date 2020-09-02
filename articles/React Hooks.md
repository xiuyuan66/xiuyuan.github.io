

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

上面代码可以看出，每个`React`组件初始化时，`DOM`都会渲染一次，`useEffect`在渲染结束后执行，如果 `useEffect` 第二个参数数组内的值发生了变化，那么 `useEffect` 第一个参数的回调将会被再执行一遍。

如果副作用存在返回函数，那么返回的函数将在卸载时运行。如果第二个参数为空数组时，不管其他状态怎么变化，此时`useEffect`都不会再次执行，接下来让我们来实践一下：
```javascript
  import React, { useState, useEffect } from 'react';
  import { Button, message } from "antd";
  let timer = null;
  const Component = ({ visible }) => {
    const [count, setCount] = useState(0);
    useEffect(() => {
      timer = setInterval(() => {
        // events ...
        message.info(`组件挂载`);
      }, 1000);
      message.info(`执行一次`);
      return () => {
        // 类似 componentWillUnmount
        // unmount events ...
        message.info(`组件卸载`);
        clearInterval(timer); // 组件卸载 移除计时器
      };
    }, []);
    return (
      <div>
        <div>{count}</div>
        <Button onClick={() => setCount(count + 1)}>click</Button>
      </div>
    );
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
![img](https://github.com/workerxuan/workerxuan.github.io/blob/master/assets/react/effect.gif?raw=true)

如果第二个参数为空，那么在每一个 State 变化时都会执行该副作用,注意这里说的参数为空是说不传递第二个参数，而不是上面说的参数为空数组

```javascript
const Components = () => {
  const [count, setCount] = useState(0);
  useEffect(() => {
    message.info(`每次状态更新,最新值${count}`);
  });
  return (
    <div>
      <div>{count}</div>
      <Button onClick={() => setCount(count + 1)}>click</Button>
    </div>
  );
};
```
![img](https://github.com/workerxuan/workerxuan.github.io/blob/master/assets/react/effectEmpty.gif?raw=true)
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
`useMemo` 类似于 `useCallback`，`useMemo`缓存返回值。它接收两个参数，第一个参数为有返回值的函数（返回值可以是任何类型），第二个参数是依赖项(数组)，当依赖项中某一个发生变化，结果将会重新调用函数生成新的返回值。

下面直接举个例子：

```javascript
  const Demo =()=> {
    const [count1, setCount1] = useState(0);
    const [count2, setCount2] = useState(10);

    const calculateCount = useMemo(() => {
      message.info('重新计算结果');
      let sum = 0;
      for (let i = 0; i < count1 * 10; i++) {
          sum += i;
      }
      return sum;

    }, [count1]);
    return (
      <div>
        {calculateCount}
        <Button onClick={() => { setCount1(count1 + 1); }}>改变count1</Button>
        <Button onClick={() => { setCount2(count2 + 1); }}>改变count2</Button>
      </div>
    );
  }

```
![img](https://github.com/workerxuan/workerxuan.github.io/blob/master/assets/react/memo1.gif?raw=true)

可以看到useMemo 与 useCallback 很像，只有依赖值改变才会重新调用函数，useMemo 也能针对传入子组件的值进行缓存优化。前面说过返回值可以是任何类型，因此，如果我们将函数的返回值替换为一个组件，那么就可以实现对组件挂载/重新挂载的性能优化。

```javascript
  const Child = ({ val }) => {
    return (
      <>
        <p>当前值：{val}</p>
      </>
    );
  };
  const Demo = () => {
    const [count1, setCount1] = useState(0);
    const [count2, setCount2] = useState(6);

    const child = useMemo(() => {
      message.info("重新生成Child组件");
      return <Child val={count1} />;
    }, [count1]);

    return (
      <div>
        {child}
        <Button onClick={() => { setCount1(count1 + 1); }}>改变count1</Button>
        <Button onClick={() => { setCount2(count2 + 1); }}>改变count2</Button>
      </div>
    );
  };

```
![img](https://github.com/workerxuan/workerxuan.github.io/blob/master/assets/react/memo2.gif?raw=true)

## useCallback
官方文档：

`Pass an inline callback and an array of dependencies. useCallback will return a memoized version of the callback that only changes if one of the dependencies has changed.`

意思就是缓存一个函数，当其中一个依赖项更改时才更改，返回一个新的函数。

使用场景是：有一个父组件，其中包含子组件，子组件接收一个函数作为props；通常而言，如果父组件更新了，子组件也会执行更新；但是大多数场景下，更新是没有必要的，我们可以借助useCallback来返回函数，然后把这个函数作为props传递给子组件；这样，子组件就能避免不必要的更新。
```javascript

  const Child = React.memo(function ({ val, type, onChange }) {
    console.log("render...");
    message.info(`${type}组件render${val}`);

    return (
      <>
        <Button onClick={onChange}>{type}Child</Button>
      </>
    );
  });

  const Parent = () => {
    const [val1, setVal1] = useState(0);
    const [val2, setVal2] = useState(0);
    const [val3, setVal3] = useState(0);

    const onChange1 = useCallback(() => {
      setVal1(val1 + 1);
    }, [val1]);
    const onChange2 = useCallback(() => {
      setVal2(val2 + 1);
    }, [val2]);
    //普通函数
    const onChange3 = () => {
      setVal3(val3 + 1);
    }
    return (
      <>
        <Child type={"first"} val={val1} onChange={onChange1} />
        <Child type={"sec"} val={val2} onChange={onChange2} />
        <Child type={"trd"} val={val3} onChange={onChange3} />
      </>
    );
  }
```
![img](https://github.com/workerxuan/workerxuan.github.io/blob/master/assets/react/cb1.gif?raw=true)

可以看出当点击`trdChild`的时候只会更新`trdChild`，而当点击`firstChild`和`secChild`的时候，更新当前点击的组件同时会更新`trdChild`，这就表示当其他组件更新的时候会导致`trdChild`重新渲染。

注意到子组件使用了 `React.memo` 这个方法，此方法内会对 `props` 做一个浅层比较，如果 `props` 没有发生改变，则不会重新渲染此组件。

```javascript
  true === true // true
  false === false // true
  1 === 1 // true
  'a' === 'a' // true

  {} === {} // false
  [] === [] // false
  () => {} === () => {} // false

  const b = {}
  b === b // true
```
上述代码说明了为什么`trdChild`每次都更新，因为`onChange3`每次都是一个新的函数，尽管长得一样，但是引用是不一样的，而`useCallback`只有在依赖项变化的时候才会返回一个新的函数，所以`React.memo` 对比后发现对象 `props` 改变，就重新渲染了。

有人可能要问如果`useCallback`的第二个参数是个空数组会怎么样呢？下面我们实践一下：
```javascript
  const Child = React.memo(function ({ val, type, onChange }) {
    console.log("render...");
    message.info(`${type}组件render${val}`);

    return (
      <>
        <Button onClick={onChange}>{type}Child</Button>
      </>
    );
  });

  const Parent = () => {
    const [val1, setVal1] = useState(0);
    const onChange1 = useCallback(() => {
      setVal1(val1 + 1);
    }, []);
    return (
      <>
        <Child type={"first"} val={val1} onChange={onChange1} />
      </>
    );
  }

```
![img](https://github.com/workerxuan/workerxuan.github.io/blob/master/assets/react/cb2.gif?raw=true)

可以看到当`useCallback`的依赖参数为一个空数组，代表着这个方法没有依赖值，将不会被更新。且由于`useCallback`自带闭包，函数内`val1`一直都是`0`。

当点击多次`firstChild`时，会发现组件只更新一次，就是因为函数内的`val1`永远都是0，所以`setVal1(0 + 1)`,`props`每次接收的值都是1，而`onChange1`没有依赖项不会返回新的函数，`Child`只会因为val1改变而更新一次继而不会再重新渲染。
