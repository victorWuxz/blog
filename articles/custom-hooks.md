# 引言
自定义hooks是react16.8版本引入hooks后一种全新的逻辑复用方式，相比render props和高阶组件有很大的优势！

本文将通过分析一个优秀的自定义Hooks库的源码来帮助读者理解自定义Hooks。

Umi Hooks 是一个 React Hooks 库，致力提供常用且高质量的 自定义Hooks。

> 阅读本文需要掌握一定的react hooks基础，还没掌握的同学需要抓紧去[官网](https://react-1251415695.cos-website.ap-chengdu.myqcloud.com/docs/hooks-intro.html)学习了。

> 除此之外还需要了解部分Umi Hooks的用法，本文主要讲解Umi Hooks中的`useRequest`、`usePrevious`、`useDebounceFn`、`useDebounce`、`useThrottleFn`、`useThrottle`、`useUpdateEffect`、`usePersistFn`,上述自定义hooks的用法还不了解的同学需要去[umi/Hooks官方文档](https://hooks.umijs.org/zh-CN/docs/getting-started)查看

> 本文的源码解析内容大部分都写到了代码注释中。

# usePersistFn

因为useRequest中使用了此hooks，我们优先讲解这个自定义Hook。
## 使用简介
它是一个持久化 function 的 Hook，通过 usePersistFn，**可以保证函数地址永远不会变化**，基本用法如下：

```js
const [count, setCount] = useState(0);

  const showCountPersistFn = usePersistFn(() => {
    message.info(`Current count is ${count}`);
  });
```
## 源码解析
基本原理是使用`useRef`和`useCallback`实现，源码如下：
```js
import {useRef, useCallback} from 'react';

// 持久化 function 的 Hook,保证函数地址永远不会变化
export default function usePersistFn(fn) {
    const ref = useRef(() => {
        throw new Error('Cannot call function while rendering.');
    });
    // 将传入的fn存储到ref中
    ref.current = fn;
    // 因为useRef创建的对象ref在函数重新渲染时地址不会改变，所以persistFn将持久化存储。
    const persistFn = useCallback(((...args) => ref.current(...args)), [ref]);
    return persistFn;
}

```

# useRequest

useRequest是一个强大的管理异步数据请求的 Hook。

## 使用简介
```js
const getUsername = params => {
    return fetch('/api/userName/get', params).then(res => res.json())
}
// gerUserName必须是一个异步函数，返回一个promise，可以带参数。
const { data, error, loading, run } = useRequest(getUsername, {
    manual: true, // 是否手动执行
    cacheKey: 'name' //如果设置了，将开启swr功能,
    debounceInterval: 500, // 如果传递了则开启防抖功能
    // 还有很多配置，不一一列举了
})
```

具体的使用方法请查阅[umi/Hooks官方文档](https://hooks.umijs.org/zh-CN/docs/getting-started)

从上述代码我们就可以感觉到它的强大，可以直接返回loading和data（意味着组件内部不用在维护loading和data），可以手动触发，有防抖节流等功能，下面我们将讲解一下它的内部实现。


## useRequest(基本版)
我们先实现一个简版的useFetch,只有发送请求返回data和loading，可以手动执行等功能：

```js
import {useEffect, useState, useCallback} from 'react';

export default function useFetch(fetch, params) {
    const [data, setData] = useState({});
    const [loading, setLoading] = useState(false);
    // 将初始的params存起来，当setNewParams的时候此hooks将重新执行。
    const [newParams, setNewParams] = useState(params);
    
    // 发送请求的核心函数，如果fetch和newParams改变重新定义
    const fetchApi = useCallback(async () => {
        setLoading(true);
        const res = await fetch(newParams);
        // 获取完数据之后调用setData和setLoading触发更新，返回新的数据
        setData(res);
        setLoading(false);
    }, [fetch, newParams]);

    // 首次默认执行一次，当组件重新渲染并且fetchApi改变时也会执行。
    useEffect(() => {
        fetchApi();
    }, [fetchApi]);

    // 手动执行函数，当调用此函数，newParams将会改变，组件重新渲染，
    // 然后fetchApi因为依赖newParams也会改变。
    // 组件渲染完之后依赖fetchApi的useEffect将会执行，从而重新调取接口获取数据。
    const run = useCallback(rest => {
        setNewParams(rest);
    }, []);

    return {
        loading,
        data,
        run,
    };
}
```

## useRequest(进阶版)

上述封装的`useFetch`已经能够满足大部分业务场景,加下来我们封装一个基本的`useRequest`(在此基础上添加防抖、节流功能、是否手动执行等功能)

```js
import {useRef, useEffect, useState, useCallback} from 'react';
import debounce from 'lodash/debounce';
import throttle from 'lodash/throttle';
import usePersistFn from './usePersistFn';

const DEFAULT_KEY = 'USE_API_DEFAULT_KEY';

// 自己封装的Fetch类，并不是js自带的fetch
class Fetch {
    that = this;
    // 请求时序，这个count主要用于处理一个页面使用多个useRequest的情况
    count = 0;

    state = {
        loading: false,
        data: undefined,
        error: undefined,
        parmas: [],
        run: this.run.bind(this.that),
    };

    constructor(service, config, subscribe) {
        this.service = service;
        this.config = config;
        this.subscribe = subscribe;
        // 如果配置了节流和防抖，使用lodash的节流防抖函数包装执行函数run
        this.debounceRun = this.config.debounceInterval ? debounce(this._run, this.config.debounceInterval) : undefined;
        this.throttleRun = this.config.throttleInterval ? throttle(this._run, this.config.throttleInterval) : undefined;
    }

    setState(s = {}) {
        this.state = {
            ...this.state,
            ...s
        };
        // 重要，改变状态的时候触发订阅，触发重新视图渲染
        // 比如获取数据返回后重置了loading，data等
        this.subscribe(this.state);
    }

    // 手动执行函数，返回一个promise，在service 返回值后后重置自身状态并触发订阅
    _run(...args) {
        this.count += 1;
        // 闭包存储当次请求的 count
        const currentCount = this.count;
        this.setState({
            loading: true,
            params: args
        });
        return this.service(...args).then(data => {
            if (currentCount === this.count) {
                this.setState({
                    data,
                    error: undefined,
                    loading: false
                });
                // 如果配置了成功的回调则调用成功的回调
                if (this.config.onSuccess) {
                    this.config.onSuccess(data, args);
                }
                return data;
            }
        })
            .catch(error => {
                if (currentCount === this.count) {
                    this.setState({
                        data: undefined,
                        error,
                        loading: false
                    });
                    // 如果配置了失败的回调则调用成功的回调
                    if (this.config.onError) {
                        this.config.onError(error, args);
                    }
                    return error;
                }
            });
    }

    // 此处添加一个run主要为了处理节流和防抖
    run(...args) {
        if (this.debounceRun) {
            this.debounceRun(...args);
            // 如果 options 存在 debounceInterval，或 throttleInterval，则 run不会返回 Promise。
            return;
        }
        if (this.throttleRun) {
            this.throttleRun(...args);
            return;
        }
        return this._run(...args);
    }
}
// 接收一个promise(service请求)和配置信息（手动执行，节流防抖等），返回data,pager，loading等信息
export default function useRequest(service, options) {

    const _options = options || {};
    const {
        manual = false,
        defaultParams = [],
        onSuccess = () => {},
        onError = () => {},
        debounceInterval,
        throttleInterval,
    } = _options;
    const newstFetchKey = useRef(DEFAULT_KEY);

    // 持久化一些函数
    // 当前请求
    const servicePersist = usePersistFn(service);
    // 成功的回调
    const onSuccessPersist = usePersistFn(onSuccess);
    // 失败的回调
    const onErrorPersist = usePersistFn(onError);

    // Fetch实例需要的配置
    const config = {
        onSuccess: onSuccessPersist,
        onError: onErrorPersist,
        debounceInterval,
        throttleInterval,
    };

    // 初始化当前的hooks
    const [fetches, setFeches] = useState(() => []);

    // 订阅函数,每次被触发都会触发函数的执行。
    const subscribe = usePersistFn((key, data) => {
        setFeches(s => {
            s[key] = data;
            return {...s};
        });
    }, []);

    // 将所有fetch请求存到ref中
    const fetchesRef = useRef(fetches);
    fetchesRef.current = fetches;

    // 手动执行函数
    const run = useCallback((...args) => {
        const currentFetchKey = newstFetchKey.current;
        let currentFetch = fetchesRef.current[currentFetchKey];
        // 如果没有已经存储的请求状态，新建一个Fetch实例并存储它的状态
        if (!currentFetch) {
            const newFetch = new Fetch(
                servicePersist,
                config,
                subscribe.bind(null, currentFetchKey),
            );
            currentFetch = newFetch.state;
            setFeches(s => {
                s[currentFetchKey] = currentFetch;
                return {...s};
            });
        }
        // 返回并执行当前Fetch实例的run函数
        return currentFetch.run(...args);
    }, [subscribe]);

    useEffect(() => {
        // 如果不是手动执行，默认请求一次
        if (!manual) {
            // 第一次默认执行，可以通过 defaultParams 设置参数
            run(...defaultParams);
        }
    }, []);

    return {
        loading: !manual,
        data: undefined,
        error: undefined,
        ...(fetches[newstFetchKey.current] || {}),
        run,
    };
}

```
上述代码和前面封装的useFetch最大的区别就是我们自己定义了一个Fetch类，每次调用run的时候会调用fetch实例的run函数，在实例的run函数中做了节流和防抖的处理，并且会触发我们自定义hooks的setFeches从而触发视图更新。

> 我们自定义一个Fetch类的好处就是可以扩展很多功能，其中就包括已经实现的节流、防抖、成功和失败的回调、格式化结果，快速改变返回数据，取消请求、屏幕聚焦重新请求等功能。

## useRequest(增加SWR能力)

上面封装的userequset已经足够满足日常业务需求了，我们再来增强一些功能，比如`SWR`(stale-while-revalidate)的能力。

> 使用方法很简单，只要在options中传入一个cacheKey参数就可以。

代码如下：

```js
import {useRef, useEffect, useState, useCallback} from 'react';
import debounce from 'lodash/debounce';
import throttle from 'lodash/throttle';
import usePersistFn from './usePersistFn';
// 实现swr的缓存函数,代码在下面
import {getCache, setCache} from './cache';

const DEFAULT_KEY = 'USE_API_DEFAULT_KEY';

class Fetch {
    that = this;

    // 请求时序
    count = 0;

    state = {
        loading: false,
        data: undefined,
        error: undefined,
        parmas: [],
        run: this.run.bind(this.that),
        refresh: this.refresh.bind(this.that),
    };

    // 增加initState参数，协助实现缓存功能
    constructor(service, config, subscribe, initState) {
        this.service = service;
        this.config = config;
        this.subscribe = subscribe;
        if (initState) {
            this.state = {
                ...this.state,
                ...initState,
            };
        }
        this.debounceRun = this.config.debounceInterval ? debounce(this._run, this.config.debounceInterval) : undefined;
        this.throttleRun = this.config.throttleInterval ? throttle(this._run, this.config.throttleInterval) : undefined;
    }

    setState(s = {}) {
        this.state = {
            ...this.state,
            ...s
        };
        // 重要，改变状态的时候触发订阅，触发hooks的重新加载
        this.subscribe(this.state);
    }

    _run(...args) {
        this.count += 1;
        // 闭包存储当次请求的 count
        const currentCount = this.count;
        this.setState({
            loading: true,
            params: args
        });
        return this.service(...args).then(data => {
            if (currentCount === this.count) {
                this.setState({
                    data,
                    error: undefined,
                    loading: false
                });
                if (this.config.onSuccess) {
                    this.config.onSuccess(data, args);
                }
                return data;
            }
        })
            .catch(error => {
                if (currentCount === this.count) {
                    this.setState({
                        data: undefined,
                        error,
                        loading: false
                    });
                    if (this.config.onError) {
                        this.config.onError(error, args);
                    }
                    return error;
                }
            });
    }

    run(...args) {
        if (this.debounceRun) {
            this.debounceRun(...args);
            // 如果 options 存在 debounceInterval，或 throttleInterval，则 run 和 refresh 不会返回 Promise;。
            return;
        }
        if (this.throttleRun) {
            this.throttleRun(...args);
            return;
        }
        return this._run(...args);
    }

    refresh() {
        return this.run(...this.state.params);
    }
}

// 接收一个promise(service请求)，返回data,pager，loading等信息
export default function useRequest(service, options) {

    const _options = options || {};
    const {
        manual = false,
        defaultParams = [],
        onSuccess = () => {},
        onError = () => {},
        debounceInterval,
        throttleInterval,
        cacheKey,
    } = _options;
    const newstFetchKey = useRef(DEFAULT_KEY);

    // 持久化一些函数
    const servicePersist = usePersistFn(service);
    const onSuccessPersist = usePersistFn(onSuccess);
    const onErrorPersist = usePersistFn(onError);

    // Fetch需要的配置
    const config = {
        onSuccess: onSuccessPersist,
        onError: onErrorPersist,
        debounceInterval,
        throttleInterval,
    };

    // 订阅函数
    const subscribe = usePersistFn((key, data) => {
        // eslint-disable-next-line no-use-before-define
        setFeches(s => {
            s[key] = data;
            return {...s};
        });
    }, []);

    // 缓存处理重点，初始化的时候获取缓存数据
    const [fetches, setFeches] = useState(() => {
        // 如果有缓存
        if (cacheKey) {
            const cache = getCache(cacheKey);
            if (cache) {
                newstFetchKey.current = cache.newstFetchKey;
                const newFetches = {};
                Object.keys(cache.fetches).forEach(key => {
                    const cachedFetch = cache.fetches[key];
                    // 将缓存的loading，params，data等赋值到新的Fetch实例中，这样用户一进来就会显示上次的结果
                    const newFetch = new Fetch(
                        servicePersist,
                        config,
                        subscribe.bind(null, key),
                        {
                            loading: cachedFetch.loading,
                            params: cachedFetch.params,
                            data: cachedFetch.data,
                            error: cachedFetch.error
                        }
                    );
                    newFetches[key] = newFetch.state;
                });
                return newFetches;
            }
        }
        return [];
    });

    const fetchesRef = useRef(fetches);
    fetchesRef.current = fetches;

    // 手动执行函数
    const run = useCallback((...args) => {
        const currentFetchKey = newstFetchKey.current;
        let currentFetch = fetchesRef.current[currentFetchKey];
        if (!currentFetch) {
            const newFetch =  new Fetch(
                servicePersist,
                config,
                subscribe.bind(null, currentFetchKey),
            );
            currentFetch = newFetch.state;
            setFeches(s => {
                s[currentFetchKey] = currentFetch;
                return {...s};
            });
        }
        return currentFetch.run(...args);
    }, [subscribe]);

    // 缓存处理，每次setFetches都会触发，将当前的fetches缓存起来
    useEffect(() => {
        if (cacheKey) {
            setCache(cacheKey, {
                fetches,
                newstFetchKey: newstFetchKey.current
            });
        }
    }, [cacheKey, fetches]);

    useEffect(() => {
        // 如果不是手动执行，默认请求一次
        if (!manual) {
            // 如果有缓存
            if (Object.keys(fetches).length > 0) {

                /* 重新执行所有的 */
                Object.values(fetches).forEach(f => {
                    f.refresh();
                });
            }
            else {
                // 第一次默认执行，可以通过 defaultParams 设置参数
                run(...defaultParams);
            }
        }
    }, []);

    return {
        loading: !manual,
        data: undefined,
        error: undefined,
        ...(fetches[newstFetchKey.current] || {}),
        run,
    };
}

```

`setCache`和`getCache`的代码如下：

```js
const cache = {};

const setCache = (key, data) => {
    if (cache[key]) {
        clearTimeout(cache[key].timer);
    }

    // 数据在不活跃 5min 后，删除掉
    const timer = setTimeout(() => {
        delete cache[key];
    }, 5 * 60 * 1000);

    cache[key] = {
        data,
        timer
    };
};

const getCache = key => cache?.[key]?.data;

export {
    getCache,
    setCache
};
```

从上面代码的注释来看，实现swr能力非常简单，只需要在每次请求的时候将数据存储到全局的缓存对象中，在初始化的时候先从缓存中获取缓存数据渲染到页面，背后还在进行请求，请求完成后会自动覆盖缓存的结果。

关于useRequest，我们暂时只讲这些源码，其余扩展功能对很多项目不是刚需，有兴趣的同学可以去umi/hooks的github查看源码。

# useUpdateEffect
## 使用简介
只在更新阶段执行的effect，用法和useEffect一样
## 源码解析

```js
import {useEffect, useRef} from 'react';

const useUpdateEffect = (effect, deps) => {
    const isMounted = useRef(false);
    useEffect(() => {
        // 首次执行的时候isMounted.current为false，所以不会执行传入的副作用函数
        if (!isMounted.current) {
            isMounted.current = true;
        }
        // 更新的时候isMounted.current已经为true
        else {
            return effect();
        }
    }, deps);
};

export default useUpdateEffect;
```

# usePrevious
保存上一次渲染时状态的 Hook
## 使用简介

```js
const [count, setCount] = useState(0);
const previous = usePrevious(count);
```

## 源码解析
主要使用useRef来存储上一次的值

```js
import {useRef} from 'react';

// 获取上一轮的props或者state
export default function usePrevious(state, compare) {
    const prevRef = useRef();
    const curRef = useRef();

    const needUpdate = typeof compare === 'function' ? compare(curRef.current, state) : true;
    if (needUpdate) {
        prevRef.current = curRef.current;
        curRef.current = state;
    }

    return prevRef.current;
}

```
# useDebounceFn
用来处理防抖函数的 Hook。

## 使用简介

```js
export default () => {
  const [value, setValue] = useState(0);
  const { run } = useDebounceFn(() => {
    setValue(value + 1);
  }, 500);

  return (
    <div>
      <Button onClick={run}>Click fast!</Button>
    </div>
  );
};
```

## 源码解析

```js
import {useCallback, useEffect, useRef} from 'react';
import useUpdateEffect from './useUpdateEffect';

function useDebounceFn(fn, deps, wait,) {
    // 如果不传递deps，只传递时间，时间也可以放在第二个参数
    const _deps = (Array.isArray(deps) ? deps : []);
    const _wait = typeof deps === 'number' ? deps : wait || 0;
    const timer = useRef();

    const fnRef = useRef(fn);
    fnRef.current = fn;

    // 取消函数
    const cancel = useCallback(() => {
        if (timer.current) {
            clearTimeout(timer.current);
        }
    }, []);

    const run = useCallback((...args) => {
        cancel();
        timer.current = setTimeout(() => {
            fnRef.current(...args);
        }, _wait);
    }, [_wait, cancel],);

    // 只在更新阶段执行
    useUpdateEffect(() => {
        run();
        return cancel;
    }, [..._deps, run]);

    // 卸载的时候取消定时器
    useEffect(() => cancel, []);

    return {
        run,
        cancel,
    };
}

export default useDebounceFn;
```
# useDebounce
用来处理防抖值的 Hook。
## 使用简介

```js
export default () => {
  const [value, setValue] = useState();
  const debouncedValue = useDebounce(value, 500);

  return (
    <div>
      <Input
        value={value}
        onChange={e => setValue(e.target.value)}
        placeholder="Typed value"
        style={{ width: 280 }}
      />
      <p style={{ marginTop: 16 }}>DebouncedValue: {debouncedValue}</p>
    </div>
  );
};
```
## 源码解析
基于useDebounceFn实现

```js
import {useState} from 'react';
import useDebounceFn from './useDebounceFn';

function useDebounce(value, wait) {
    const [state, setState] = useState(value);
    
    //包了一层防抖的hooks，在获取value的时候会触发防抖机制。
    useDebounceFn(
        () => {
            setState(value);
        },
        [value],
        wait,
    );
    return state;
}

export default useDebounce;

```
# useThrottleFn
用来处理函数节流的 Hook。
## 使用简介

```js
export default () => {
  const [value, setValue] = useState(0);
  const { run } = useThrottleFn(() => {
    setValue(value + 1);
  }, 500);

  return (
    <div>
      <p style={{ marginTop: 16 }}> Clicked count: {value} </p>
      <Button onClick={run}>Click fast!</Button>
    </div>
  );
};

```
## 源码解析

```js
import {useCallback, useEffect, useRef} from 'react';
import useUpdateEffect from '../useUpdateEffect';

function useThrottleFn(fn, deps, wait,) {
    const _deps = (Array.isArray(deps) ? deps : []);
    const _wait = typeof deps === 'number' ? deps : wait || 0;
    const timer = useRef();

    const fnRef = useRef(fn);
    fnRef.current = fn;

    const currentArgs = useRef([]);

    const cancel = useCallback(() => {
        if (timer.current) {
            clearTimeout(timer.current);
        }
        timer.current = undefined;
    }, []);

    // 节流的处理，一定时间内只触发一次
    const run = useCallback((...args) => {
        currentArgs.current = args;
        if (!timer.current) {
            timer.current = setTimeout(() => {
                fnRef.current(...currentArgs.current);
                timer.current = undefined;
            }, _wait);
        }
    }, [_wait, cancel]);

    useUpdateEffect(() => {
        run();
    }, [..._deps, run]);

    useEffect(() => cancel, []);

    return {
        run,
        cancel,
    };
}

export default useThrottleFn;
```

# useThrottle
用来处理值节流 Hook。
## 使用简介

```js
export default () => {
  const [value, setValue] = useState();
  const throttledValue = useThrottle(value, 500);

  return (
    <div>
      <Input
        value={value}
        onChange={e => setValue(e.target.value)}
        placeholder="Typed value"
        style={{ width: 280 }}
      />
      <p style={{ marginTop: 16 }}>throttledValue: {throttledValue}</p>
    </div>
  );
};
```
## 源码解析
基于useThrottleFn实现

```js
import {useState} from 'react';
import useThrottleFn from './useThrottleFn';

function useThrottle(value, wait) {
    const [state, setState] = useState(value);
    useThrottleFn(
        () => {
            setState(value);
        },
        [value],
        wait,
    );
    return state;
}
export default useThrottle;

```

# 总结

* 自定义hooks可以极大地提升我们开发效率。
* 灵活运用useRef，useCallback，useEffect等基本hook可以实现很多高质量自定义hook。
* 在自定义hooks中如果调用了"setState"或者"dispatch"就会触发整个函数组件的更新，从而能获取到自定义hook中处理后的最新的数据。
* hooks让swr的实现变得非常简单，目前优质的swr自定义hooks有本文讲的useRequest和github上star数量很多的useSwr。


# 参考文献

* [React Hooks](https://react-1251415695.cos-website.ap-chengdu.myqcloud.com/docs/hooks-intro.html)
* [Umi Hooks](https://hooks.umijs.org/zh-CN/docs/getting-started)