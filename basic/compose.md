# 对比koa、redux、axios中间件系统

## 这些中间件干了什么？
简言之，大部分前端框架的中间件都是把某些逻辑（比如，日志、数据序列化等）抽离出去，中间件系统就是在数据流转过程中增加一个处理数据的口子。
从形式上讲，大部分中间件系统都开放一个注册中间件的API，使用者把中间件注册进去即可正常使用。每一个中间件本身一般是一个函数，这个函数通常可以接受`context`和`next`两个参数，用户在中间件中可以通过`context`拿到前面中间件处理后的数据，又可以通过`next`调用下一个中间件。
类似于
```javascript
function middleware1 (context, next) { // 中间件1
  useContextHere(context);
  ...
  next(); // call next middleware
}
function middleware2 (context, next) { // 中间件2
  ...
}
function middleware3 (context, next) { // 中间件3
  ...
}
// 注册中间件
instance.registe(middleware1);
instance.registe(middleware2);
instance.registe(middleware3);
// 或者直接注册多个
instance.registe([middleware1, middleware2, middleware3]);
```

# 实现一个简单的中间件系统
先写几个中间件
```js
function f1 (ctx, next) {
  ctx.name = 'javascript';
  next();
}

function f2 (ctx, next) {
  ctx.greet = 'hello'; 
  next();   
}

function f3 (ctx) {
  ctx.echo = () => console.log(`${ctx.greet} ${ctx.name}!`);
  ctx.echo();
}
```
接下来是实现`registe`API
```js
class Proto {
  constructor() {
    this.ctx = {};
    this.middles = []; 
  }

  registe = (middles) => {
    this.middles = [...this.middles, ...middles];
  }

  execute = () => {
    compose(this.middles)(this.ctx);
  };
}

function compose (middles) {
  return ctx => {
    let length = middles.length;
    let i = 0;
    function next () {
      if (i <= length - 1) {
        let fn = middles[i];
        i++;
        fn(ctx, next);
      }
    }
    next(); 
  };
}

const instance = new Proto();
instance.registe([f1, f2, f3]);
instance.execute();
```
可以看到，中间件系统的一个核心函数是`compose`，这个函数采用不同的实现可以得到不同的中间件系统。

## koa中间件
koa中间件整体思路和上面类似，稍有区别的是koa2.0版本开始全面支持`async await`的写法，实际使用如下
```js
app.use(async function ( ctx, next ) {
    xxx(ctx);
    await next()
  }
});
```
因为中间件可能是一个`async`函数，所以`koa-compose`函数把每次调用都包装成返回一个`promise`，具体实现如下：
```js
function compose (middleware) {
  if (!Array.isArray(middleware)) throw new TypeError('Middleware stack must be an array!')
  for (const fn of middleware) {
    if (typeof fn !== 'function') throw new TypeError('Middleware must be composed of functions!')
  }
  return function (context, next) { 
    // last called middleware #
    let index = -1
    return dispatch(0)
    function dispatch (i) {
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i
      let fn = middleware[i] // 取出函数
      if (i === middleware.length) fn = next // 处理边界
      if (!fn) return Promise.resolve() 
      try { // 下一个中间件调用都被promise包裹
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
      } catch (err) {
        return Promise.reject(err)
      }
    }
  }
}
```

## redux
redux的中间件理解起来要复杂一点，要理解redux函数的原理关键点在于理解redux中间件系统中流转的数据是什么。

eg：redux有一个原则，ruducer都是纯函数，所以不经过改造是不能处理异步场景的
redux默认的使用方法如下：
```js
store.dispatch({ action, payload });

function reducer ({action, payload}, state) {
  switch () {
    case 'action1': 
      doSomething();
      return newState;
      break;
    case 'otherAction':
      ...
  }
}
```
如果要做异步操作，需要把一个action拆成多个action，类似于
```js
store.strengthenDispatch({action: 'fetchAPI', payload}) = store.dispatch({action: 'startLoad', payload}) + store.dispatch({action: 'afterLoad', payload});
```
所以redux中间件系统就是改造store.dispatch方法，每个中间件改造和传递的都是store.dispatch
看一个最简单的日志中间需求，每次dispatch之前打印一下日志，大致功能如下
```js
const logger = store => {
  return action => {
    console.log('logger info');
    store.dispatch(action);
    console.log(store.getState());
  };
};
```
上文说过，用户使用的时候都是调用`store.dispatch`，同时还可能存在别的中间件，中间件函数必须能拿到`store.dispatch`，也能把`store.dispatch`传给下一个中间件，所以一个标准中间件会长这样
```js
const logger = store => {
  return next => { // next是前一个中间件传过来的dispatch
    return action => {
      console.log('logger info');
      next(action);
      console.log(store.getState());
    };
  }
};
```

了解了redux中间件就能明白redux组合中间件的目标是什么，看一下源码
```ts
export default function applyMiddleware(
  ...middlewares: Middleware[]
): StoreEnhancer<any> {
  return (createStore: StoreEnhancerStoreCreator) => <S, A extends AnyAction>(
    reducer: Reducer<S, A>,
    preloadedState?: PreloadedState<S>
  ) => {
    const store = createStore(reducer, preloadedState)
    let dispatch: Dispatch = () => {
      throw new Error(
        'Dispatching while constructing your middleware is not allowed. ' +
          'Other middleware would not be applied to this dispatch.'
      )
    }

    const middlewareAPI: MiddlewareAPI = {
      getState: store.getState,
      dispatch: (action, ...args) => dispatch(action, ...args)
    }
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose<typeof dispatch>(...chain)(store.dispatch) // 核心方法

    return {
      ...store,
      dispatch
    }
  }
}

export default function compose(...funcs: Function[]) {
  if (funcs.length === 0) {
    // infer the argument type so it is usable in inference down the line
    return <T>(arg: T) => arg
  }

  if (funcs.length === 1) {
    return funcs[0]
  }

  return funcs.reduce((a, b) => (...args: any) => a(b(...args)))
}
```
其中，这行代码比较难理解
```js
funcs.reduce((a, b) => (...args: any) => a(b(...args)))
```
实际效果举例说明就是
假设有多个中间件`f1 f2 f3 f4`，经过compose的结果就是`f1(f2(f3(f4(...args))))`，执行结果就是
```js
dispatch = f1(f2(f3(f4(...args))))(store.dispatch)
```
再具体一些，代入我们上文提到的logger中间件说明，经过下面这两行代码，得到的chain中每一项是中间件的第二层函数
```js
const middlewareAPI: MiddlewareAPI = {
      getState: store.getState,
      dispatch: (action, ...args) => dispatch(action, ...args)
    }
const chain = middlewares.map(middleware => middleware(middlewareAPI))
// 标准中间件 middleware = store => next => action => {...}
// 执行middleware(middlewareAPI)，这里的middlewareAPI就是被当作store参数传入，看middlewareAPI也能明白，这个对象就是一个简化版的store
// 所以经过map，chain中的每一项是 next => action => {...} 这样一个高阶函数
```
再继续看下一行代码的返回结果
```js
dispatch = compose<typeof dispatch>(...chain)(store.dispatch)
```
先看最简单的情况，如果只有一个中间件，那么compose函数返回的就是chain中的原始元素，所以得到
`dispatch = (next => action => {...})(store.dispatch) = (action => { ...这里如果调用了next就是调用原始的store.dispatch })`
```js
function compose(...funcs: Function[]) {
  ...
  if (funcs.length === 1) {
    return funcs[0]
  }
  ...
}
```
再看多个中间件的情况，其实和一个中间件没有什么差别
首先`chain`是这样的`[next1 => action1 => {...}, next2 => action2 => {...}, next3 => action3 => {...}, ...]`,
注意，这里的`action1/2/3`都是行参，取名不同不一定代表实际不同
把这个代入到我们前面的结论中
```js
dispatch = f1(f2(f3(f4(...args))))(store.dispatch)
```
得到
```js
dispatch = (((next3 => action3 => {...}) => action2 => {...}) => action1 => {...}))(store.dispatch)
// 运行结果就是store.dispatch被当作参数传给了next3 => action3 => {...}，这其实回到了前面只有一个中间件的情形，
// (next3 => action3 => {...})(store.dispatch) 得到 (action3 => {...这里的dispatch就是store.dispatch})
```
所以当使用者`store.dispatch(action)`时，`action`会首先传递到第3个中间件，但是第3个中间件的内层不会执行，会作为整体传给第2个中间件，同样套娃，第2个中间件会把前面的整体传给第1个中间件
此时实际情形如下
中间件3中：`next === store.disptach`
中间件2中：`next === (action => {...}) // 整个中间件3`
中间件1中：`next === ((action => {...}) => action => {...}) // 整个中间件2，其中中间件2的参数部分是中间件3`
这就保证了整个模式还是洋葱模型，依照1->2->3顺序调用，当中间件内部调用`next`时会触发下一个中间件，最后一个中间件中next就是`store.dispatch`，然后再按照3->2->1的顺序回来。当然，这里有个问题，如果某个中间件中没调用next，那整个流程就断了。

## axios
axios的中间件系统比较容易理解，整体看axios分成两类中间件，一类在发起请求前起作用，一类在发起请求后起作用。
因为axios核心就是个请求库，可以简单理解为
```js
axios(config) = Promise.resolve(config);  // 整个库的核心就是Axios.prototype.request方法下的这行代码
```
axios可以注册拦截器，这些拦截器的中间件都被放在一个`chain`中，上面的核心函数使整个chain的中间点，所有request阶段的拦截器放在前面，response阶段的拦截器放在后面
```js
this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
  chain.unshift(interceptor.fulfilled, interceptor.rejected);
});
this.interceptors.response.forEach(function unshiftRequestInterceptors(interceptor) {
  chain.unshift(interceptor.fulfilled, interceptor.rejected);
});

// 得到chain = [[...interceptors.request], Promise.resolve(config), [...interceptors.response]];
```
整体流程非常简单，就是一个循环，运行一个promise链，每次取出一对函数对应一个`promise`的（`resolve`、`reject`）一对参数，运行结果传递给下一个拦截器。
```js
// 请求拦截器->http请求->响应拦截器
while (chain.length) {
  promise = promise.then(chain.shift(), chain.shift());
}
```

## 总结
这三个中间件系统相似的地方都是用数组结构存储中间件，不同之处在于组合函数不同
其中符合洋葱模型的是koa和redux，axios直接通过promise的then链传递数据
koa和axios都把结果把装成promise解决异步问题，redux是增强了next方法把异步问题替换成多个分布的同步问题

