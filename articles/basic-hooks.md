# 什么是Hooks
Hook 是 React 16.8 的新增特性。它可以让你在不编写 class 的情况下使用 state 以及其他的 React 特性。

# 为什么要用Hooks
## 代码可读性好，易于维护

### 1.hooks在function组件中使用，不用维护复杂的生命周期，不用担心this指向问题
Hooks给Function组件赋能，Function组件也可维护自己的state，不用担心组件通信过程中this指向的问题。
### 2.更好的逻辑复用方式
自定义hook相比目前react常见的代码复用方式（[高阶组件](https://react.docschina.org/docs/higher-order-components.html)，[render props](http://react.docschina.org/docs/render-props.html)）都要简单易懂，具体可以参照本章[自定义hooks章节](#自定义Hooks)
## 提升开发效率

我们来对比一下同一个功能用class组件实现和使用hooks的function组件实现的代码差异,

### 1.Class组件版本
```js
import React from 'react';
class Person extends React.Component {
  constructor(props) {
      super(props);
      this.state = {
          username: "小明"
      };
  }
  
  componentDidMount() {
      console.log('组件挂载后要做的操作')
  }
  
  componentWillUnmount() {
      console.log('组件卸载要做的操作')
  }
  
  componentDidUpdate(prevProps, prevState) {
      if(prevState.username !== this.state.username) {
          console.log('组件更新后的操作')
      }
  }
  
  render() {
      return (
        <div>
            <p>欢迎 {state.username}</p>
            <input type="text" placeholder="input a username" onChange={(event) => this.setState({ username: event.target.value)})}></input>
        </div>
      );
  }
}
```
### 2.Hooks版本
```js
import React, {useState， useEffect} from 'react';

export const Person = () => {
  const [name, setName] = useState("小明");
  
  useEffect(() => {
      console.log('组件挂载后要做的操作')
      return () => {
        console.log('组件卸载要做的操作')
      }
  }, []);
  
  useEffect(() => {
      console.log('组件更新后的操作')
  }, [name]);
  
  return (
    <div>
        <p>欢迎 {name}</p>
        <input type="text" placeholder="input a username" onChange={(event) => setName( event.target.value)}></input>
    </div>
  )
}
```

Hooks版本简化了很多代码，熟悉后可以显著提升开发效率。

# 怎样使用Hooks

## Hooks基础API

### useState(重点掌握)
#### 1.参数：
* 常量：组件初始化的时候就会定义
```js
import React, { useState } from 'react';

function Example() {
  // 声明一个叫 "count" 的 state 变量,初始值为0，后续通过setCount改变它能让视图重新渲染
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}
```
* 函数：只有开始渲染的时候函数才会执行
```js
// initialState 参数只会在组件的初始渲染中起作用，后续渲染时会被忽略。
// 如果初始 state 需要通过复杂计算获得，则可以传入一个函数，在函数中计算并返回初始的 state，
// 此函数只在初始渲染时被调用：
const [count, setCount] = useState(() => {
  const initialCount = someExpensiveComputation(props);
  return initialState;
})
```

#### 2.返回值
useState返回值时一个长度为2的数组，数组第一项为为定义的变量（名称自己定），第二项时改变第一项的函数（名称自己定），具体示例可看上述代码。

### useEffect(重点掌握)
该 Hook 有两个参数，第一个参数是一个包含命令式、且可能有副作用代码的函数，第二个参数是一个数组，此参数来控制该Effect包裹的函数执不执行，**如果第二个参数不传递，则该Effect每次组件刷新都会执行，相当于class组件中的componentDidMount和componentDidupdate生命周期的融合**。

#### 1.基本使用方法
```js
import React, { useState, useEffect } from 'react';

function Example() {
  const [count, setCount] = useState(0);

  // Similar to componentDidMount and componentDidUpdate:
  useEffect(() => {
    // Update the document title using the browser API
    document.title = `You clicked ${count} times`;
  });

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}
```
#### 2.控制函数的执行
和上述代码类似，我们给useEffect传递第二个参数```[count]```，这样只有count改变的时候才会执行
```js
import React, { useState, useEffect } from 'react';

function Example() {
  const [count, setCount] = useState(0);

  // 只有count改变时才会执行
  useEffect(() => {
    // Update the document title using the browser API
    document.title = `You clicked ${count} times`;
  },[count]);

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}
```
```js
import React, { useEffect } from 'react';

function Example() {
  // 组件挂载时只执行一次
  useEffect(() => {
    console.log("只执行一次，类似componentDidMount")
  },[]);

  return (
    <div>只执行一次的Effect</div>
  );
}
```
#### 3.需要清除的副作用
有一些副作用是需要清除的。例如订阅外部数据源。这种情况下，清除工作是非常重要的，可以防止引起内存泄露！

##### 示例1(每次渲染都会清除)：
```js
import React, { useState, useEffect } from 'react';

function FriendStatus(props) {
  const [isOnline, setIsOnline] = useState(null);

  useEffect(() => {
    function handleStatusChange(status) {
      setIsOnline(status.isOnline);
    }

    ChatAPI.subscribeToFriendStatus(props.friend.id, handleStatusChange);
    // Specify how to clean up after this effect:
    return function cleanup() {
      ChatAPI.unsubscribeFromFriendStatus(props.friend.id, handleStatusChange);
    };
  });

  if (isOnline === null) {
    return 'Loading...';
  }
  return isOnline ? 'Online' : 'Offline';
}
```
##### 示例2(只有组件卸载的时候清除)：
但我们给第二个参数传递一个空数组的时候，只有组件**卸载**时，Effect才会执行清除操作，此时的useEffect相当于class组件的componentDidMount和compinentWillUnmount的融合。
```js
import React, { useState, useEffect } from 'react';

function FriendStatus(props) {
  const [isOnline, setIsOnline] = useState(null);

  useEffect(() => {
    function handleStatusChange(status) {
      setIsOnline(status.isOnline);
    }

    ChatAPI.subscribeToFriendStatus(props.friend.id, handleStatusChange);
    // Specify how to clean up after this effect:
    return function cleanup() {
      ChatAPI.unsubscribeFromFriendStatus(props.friend.id, handleStatusChange);
    };
  },[]);

  if (isOnline === null) {
    return 'Loading...';
  }
  return isOnline ? 'Online' : 'Offline';
}
```

**我们在日常使用的时候要灵活运用，但尽量使用第二个参数来控制函数的执行，这样能优化性能。**

### useContext(重要)

该Hook接收一个 context 对象（React.createContext 的返回值）并返回该 context 的当前值。当前的 context 值由上层组件中距离当前组件最近的 <MyContext.Provider> 的 value prop 决定。
#### 1.使用实例：
```js
const themes = {
  light: {
    foreground: "#000000",
    background: "#eeeeee"
  },
  dark: {
    foreground: "#ffffff",
    background: "#222222"
  }
};

// 主题context
const ThemeContext = React.createContext(themes.light);

function App() {
  // 这里的value值改变，useContext包裹的值也会改变
  return (
    <ThemeContext.Provider value={themes.dark}>
      <Toolbar />
    </ThemeContext.Provider>
  );
}

function Toolbar(props) {
  return (
    <div>
      <ThemedButton />
    </div>
  );
}

function ThemedButton() {
  // 上层最近的Provider的value属性的值
  const theme = useContext(ThemeContext);

  return (
    <button style={{ background: theme.background, color: theme.foreground }}>
      I am styled by theme context!
    </button>
  );
}
```
#### 2.Class组件实现相同的逻辑请参考[react官方文档-Context](https://react.docschina.org/docs/context.html)
简单示例：
```js
// Context 可以让我们无须明确地传遍每一个组件，就能将值深入传递进组件树。
// 为当前的 theme 创建一个 context（“light”为默认值）。
const ThemeContext = React.createContext('light');

class App extends React.Component {
  render() {
    // 使用一个 Provider 来将当前的 theme 传递给以下的组件树。
    // 无论多深，任何组件都能读取这个值。
    // 在这个例子中，我们将 “dark” 作为当前的值传递下去。
    return (
      <ThemeContext.Provider value="dark">
        <Toolbar />
      </ThemeContext.Provider>
    );
  }
}

// 中间的组件再也不必指明往下传递 theme 了。
function Toolbar(props) {
  return (
    <div>
      <ThemedButton />
    </div>
  );
}

class ThemedButton extends React.Component {
  // 指定 contextType 读取当前的 theme context。
  // React 会往上找到最近的 theme Provider，然后使用它的值。
  // 在这个例子中，当前的 theme 值为 “dark”。
  static contextType = ThemeContext;
  render() {
    return <Button theme={this.context} />;
  }
}
```

### useReducer(重要)
useState 的替代方案。它接收一个形如 (state, action) => newState 的 reducer，并返回当前的 state 以及与其配套的 dispatch 方法（和redux用法十分相近）。
```js
const [state, dispatch] = useReducer(reducer, initialArg, init);
```
在某些场景下，useReducer 会比 useState 更适用，例如 state 逻辑较复杂且包含多个子值，或者下一个 state 依赖于之前的 state 等。并且，使用 useReducer 还能给那些会触发深更新的组件做性能优化，因为你可以向子组件传递 dispatch 而不是回调函数 。

参数：
* 第一个参数是reducer纯函数
* 第二个参数是初始的state
* 第三个参数可以修改初始state，将初始 state 设置为 init(initialArg)

#### 1.基本用法
```js
const initialState = {count: 0};

function reducer(state, action) {
  switch (action.type) {
    case 'increment':
      return {count: state.count + 1};
    case 'decrement':
      return {count: state.count - 1};
    default:
      throw new Error();
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);
  return (
    <>
      Count: {state.count}
      <button onClick={() => dispatch({type: 'decrement'})}>-</button>
      <button onClick={() => dispatch({type: 'increment'})}>+</button>
    </>
  );
}
```

### useCallback(重点掌握)
把内联回调函数及依赖项数组作为参数传入 useCallback，它将返回该回调函数的 memoized 版本，**该回调函数仅在某个依赖项改变时才会更新**。
* 常见应用场景：父组件向子组件传递会回调函数（但**是react官方不推荐这种方式**，官方推荐使用useReducer hook，通过传递dispatch来避免这种形式，具体原因参考[官方解释](https://react.docschina.org/docs/hooks-faq.html#how-to-read-an-often-changing-value-from-usecallback)）
* 示例：
```js
import React, { useEffect, useState, useCallback } from 'react';
// 子组件
function Son({callback}) {
    renturn (
        <a onClick={()=>callback("小红")}>点击切换姓名</a>
    )
}
// 父组件
function Parent() {
  const [name,setName] = useState("")
  useEffect(() => {
    console.log("获取数据并更新state")
    setName("小明")
  },[]);
  const callback = useCallback(name => {
    setName(name);
  }, []);
  return (
    <>
      <Son callback={callback} />;
      name:{name}
    <>
  )
}
```
### useMemo(重点掌握)
useCallback(fn, deps) 相当于 useMemo(() => fn, deps)。


把“创建”函数和依赖项数组作为参数传入 useMemo，它仅会在某个依赖项改变时才重新计算 memoized 值。**这种优化有助于避免在每次渲染时都进行高开销的计算**。


**如果没有提供依赖项数组，useMemo 在每次渲染时都会计算新的值。**

**你可以把 useMemo 作为性能优化的手段，但不要把它当成语义上的保证!**

应用场景：
* 存储一次昂贵的计算
```js
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
```
* 跳过一次子节点的昂贵的重新渲染
```js
function Parent({ a, b }) {
  // Only re-rendered if `a` changes:
  const child1 = useMemo(() => <Child1 a={a} />, [a]);
  // Only re-rendered if `b` changes:
  const child2 = useMemo(() => <Child2 b={b} />, [b]);
  return (
    <>
      {child1}
      {child2}
    </>
  )
}
```
### useRef(重要)
useRef 返回一个**可变的** ref 对象，其 **current** 属性被初始化为传入的参数（initialValue）。**返回的 ref 对象在组件的整个生命周期内保持不变**。
```js
const refContainer = useRef(initialValue);
```
使用场景：
* 访问子组件dom
```js
function TextInputWithFocusButton() {
  const inputEl = useRef(null);
  const onButtonClick = () => {
    // `current` 指向已挂载到 DOM 上的文本输入元素
    inputEl.current.focus();
  };
  return (
    <>
      <input ref={inputEl} type="text" />
      <button onClick={onButtonClick}>Focus the input</button>
    </>
  );
}
```
* 保存实例变量
```js
function Timer() {
  const intervalRef = useRef();

  useEffect(() => {
    const id = setInterval(() => {
      // ...
    });
    intervalRef.current = id;
    return () => {
      clearInterval(intervalRef.current);
    };
  });
  // ...
  return <div>使用useRef存储实例变量</div>
}
```
### useImperativeHandle(不常用)
```js
useImperativeHandle(ref, createHandle, [deps])
```
useImperativeHandle 可以让你在使用 ref 时自定义暴露给父组件的实例值。在大多数情况下，应当避免使用 ref 这样的命令式代码。useImperativeHandle 应当与 forwardRef 一起使用：
```js
function FancyInput(props, ref) {
  const inputRef = useRef();
  useImperativeHandle(ref, () => ({
    focus: () => {
      inputRef.current.focus();
    }
  }));
  return <input ref={inputRef} ... />;
}
FancyInput = forwardRef(FancyInput);
```
在本例中，渲染 ```<FancyInput ref={inputRef} />``` 的父组件可以调用 ```inputRef.current.focus()```。

### useLayoutEffect(不常用)
其函数签名与 useEffect 相同，使用方法一致，但它会在所有的 DOM 变更之后同步调用 effect。可以使用它来读取 DOM 布局并同步触发重渲染。在浏览器执行绘制之前，useLayoutEffect 内部的更新计划将被同步刷新。

尽可能使用标准的 useEffect 以避免阻塞视觉更新。

* useEffect与 componentDidMount、componentDidUpdate 不同的是，在浏览器完成布局与绘制之后，传给 useEffect 的函数会延迟调用。
* useLayoutEffect则与componentDidMount、componentDidUpdate调用时机相同。
### useDebugValue(不常用)

开发阶段调试时使用，具体用法参考[官方文档](https://react.docschina.org/docs/hooks-reference.html#usedebugvalue)

## Hook进阶

### 自定义Hooks<a name="自定义Hooks"></a>

通过自定义 Hook，可以将**抽取多个组件可重用的逻辑**，实现逻辑复用。

示例(以下示例出自[阮一峰的网络日志](https://www.ruanyifeng.com/blog/2019/09/react-hooks.html))：
```js
const Person = ({ personId }) => {
  const [loading, setLoading] = useState(true);
  const [person, setPerson] = useState({});

  useEffect(() => {
    setLoading(true); 
    fetch(`https://swapi.co/api/people/${personId}/`)
      .then(response => response.json())
      .then(data => {
        setPerson(data);
        setLoading(false);
      });
  }, [personId])

  if (loading === true) {
    return <p>Loading ...</p>
  }

  return <div>
    <p>You're viewing: {person.name}</p>
    <p>Height: {person.height}</p>
    <p>Mass: {person.mass}</p>
  </div>
}
```
我们将上述代码中获取person的逻辑抽离出来，方便其他类似的组件调用
```js
const usePerson = (personId) => {
  const [loading, setLoading] = useState(true);
  const [person, setPerson] = useState({});
  useEffect(() => {
    setLoading(true);
    fetch(`https://swapi.co/api/people/${personId}/`)
      .then(response => response.json())
      .then(data => {
        setPerson(data);
        setLoading(false);
      });
  }, [personId]);  
  return [loading, person];
};
```
上述代码中的usePerson就是一个自定义hook，在其余组件中我们可以这样使用：
```js
const Person = ({ personId }) => {
  const [loading, person] = usePerson(personId);

  if (loading === true) {
    return <p>Loading ...</p>;
  }

  return (
    <div>
      <p>You're viewing: {person.name}</p>
      <p>Height: {person.height}</p>
      <p>Mass: {person.mass}</p>
    </div>
  );
};
```

#### 自己动手实现几个常用自定义hooks

* useFetch(简单版):获取接口数据
```js
import { useState, useEffect} from 'react';
import fetch from 'fetch';

/**
 * @param {String} url 
 * @param {Object} initState 
 */
const useFetch_0 = (url, initState) => {
  const [isLoading, setIsLoading] = useState(false);
  const [data, setDate] = useState(initState);
  const [isError, setIsError] = useState(false);

  useEffect(() => {
    const fetchData = async () =>{
      setIsLoading(true);
      try {
        const res = await fetch(url);
        setDate(res);
      } catch (error) {
        setIsError(true);
      }
      setIsLoading(false);
    }
    fetchData();

  }, [url]);

  return [
    data,
    isLoading,
    isError,
  ];
}

export default useFetch_0;
```
父页面使用：```const [data,isLoading,isError] = useFetch(url,initState)```
* usePrevious:获取上一轮的props和state
```js
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => {
    ref.current = value;
  });
  return ref.current;
}

// 使用
function Counter() {
  const [count, setCount] = useState(0);
  const prevCount = usePrevious(count);
  return <h1>Now: {count}, before: {prevCount}</h1>;
}
```

#### 第三方优质自定义Hooks

github目前已经有很多优质自定义hooks，参考地址：https://github.com/rehooks/awesome-react-hooks

##### 自定义hooks举例

* **[use-deep-compare](https://github.com/sandiiarov/use-deep-compare)**

**useDeepCompareEffect**
```js
import React from 'react';
import { useDeepCompareEffect } from 'use-deep-compare';

function App({ object, array }) {
  useDeepCompareEffect(() => {
    // do something significant here
    return () => {
      // return to clean up that significant thing
    };
  }, [object, array]);

  return <div>{/* render significant thing */}</div>;
}
```
**useDeepCompareCallback**
```js
import React from 'react';
import { useDeepCompareCallback } from 'use-deep-compare';

function App({ object, array }) {
  const callback = useDeepCompareCallback(() => {
    // do something significant here
  }, [object, array]);

  return <div>{/* render significant thing */}</div>;
}
```
**useDeepCompareMemo**
```js
import React from 'react';
import { useDeepCompareMemo } from 'use-deep-compare';

function App({ object, array }) {
  const memoized = useDeepCompareMemo(() => {
    // do something significant here
  }, [object, array]);

  return <div>{/* render significant thing */}</div>;
}
```

* **[use-debounce](https://github.com/xnimorz/use-debounce)**
```js
import React, { useState } from 'react';
import { useDebounce } from 'use-debounce';

export default function Input() {
  const [text, setText] = useState('Hello');
  const [value] = useDebounce(text, 1000);

  return (
    <div>
      <input
        defaultValue={'Hello'}
        onChange={(e) => {
          setText(e.target.value);
        }}
      />
      <p>Actual value: {text}</p>
      <p>Debounce value: {value}</p>
    </div>
  );
}
```
* **[use-async-memo](https://github.com/awmleer/use-async-memo)**
```js
const data = useAsyncMemo(doAPIRequest, [])
```

### 使用Hooks实现Class组件常用生命周期
* **componentDidMount**
```js
useEffect(()=>{
    // do something
},[])
```
* **componentDidUpdate**
```js
useEffect(()=>{
    // do something
})
```
* **componentWillUnmount**
```js
useEffect(()=>{
    return ()=> {
        // do something
    }
},[])
```
* **getDerivedStateFromProps:[官方教程](https://react-1251415695.cos-website.ap-chengdu.myqcloud.com/docs/hooks-faq.html#how-do-i-implement-getderivedstatefromprops)**
```js
function ScrollView({row}) {
  let [isScrollingDown, setIsScrollingDown] = useState(false);
  let [prevRow, setPrevRow] = useState(null);

  if (row !== prevRow) {
    // Row 自上次渲染以来发生过改变。更新 isScrollingDown。
    setIsScrollingDown(prevRow !== null && row > prevRow);
    setPrevRow(row);
  }

  return `Scrolling down: ${isScrollingDown}`;
}
```

* **shouldComponentUpdate**


**可以使用useMemo，如果不涉及比较组件内部state，建议使用memo**
```js
function Parent({ a, b }) {
  // Only re-rendered if `a` changes:
  const child1 = useMemo(() => <Child1 a={a} />, [a]);
  // Only re-rendered if `b` changes:
  const child2 = useMemo(() => <Child2 b={b} />, [b]);
  return (
    <>
      {child1}
      {child2}
    </>
  )
}
```

## Hooks常见问题

大部分常见的问题在上述代码中都体现了，其余问题请参考[官方文档问题模块](https://react-1251415695.cos-website.ap-chengdu.myqcloud.com/docs/hooks-faq.html)

## Hooks注意事项

* 只在最顶层使用 Hook
* 只在 React 函数中调用 Hook
* 详细规则请参考官方文档[hooks规则](https://react-1251415695.cos-website.ap-chengdu.myqcloud.com/docs/hooks-rules.html)

# 总结

* useState和useEffect可以覆盖绝大多数业务场景
* 复杂的组件使用useReducer代替useState
* 在useState和useEffect不满足业务需求的时候，使用useContext，useRef，或者第三方自定义钩子来解决
* useMemo和useCallback用来做性能优化，如果不用他俩代码应该也能正确运行

# 参考文献

* [React Hooks官方文档](https://react-1251415695.cos-website.ap-chengdu.myqcloud.com/docs/hooks-intro.html)
* [阮一峰的网络日志之Hooks入门教程](http://www.ruanyifeng.com/blog/2019/09/react-hooks.html)
