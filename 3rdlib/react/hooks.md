*[React Hooks文档地址](https://zh-hans.reactjs.org/docs/hooks-reference.html#usecallback)*

记录一下React Hooks相关知识点

## Function Component
所有的Hooks都只能在函数式组件中使用，所以理解Hooks的第一步是对函数式组件有正确的认识。
有种“误区”认识是将Function Component和class Component类比比对，或许这种方式能最快速上手函数式组件，但是理解越深入就越能明白这两者不完全是同一纬度的替换。具体内容不展开讲，有足够深入认识后单独写一篇。

## useState和useReducer
在React v16.8之前的版本里函数式组件是没有状态的，所以又叫木偶组件(Dumb Component)/无状态组件（Stateless Component）等等，有了Hooks才让函数式组件可以有`state`，也因为加了`state`，基本可以认为函数式组件能实现class Component的所有功能。

### 从useState入门
所以最基础的Hooks就是useState
```js
function Fc(props) {
  const [ x, setX ] = useState(initialValue);

  doSonmthing() {
    setX(updateValue);
  }

  return ...;
}
```
上面的组件就有了x这个state，且可以更新。
这里应该有两个令人疑惑的点：
  1. 单独看这个函数，每次渲染进入`Fc`函数体都会执行`useState`返回x，那为什么不是每一次x都被重置成初始值？
    这里就要理解组件在初始化（mount）之后，所有state都被存到了对应的Fiber Node中。
    在`fiber.memoizedState`中保存了`hook`这样一个数据结构，同时是每一个`useState`都对应一个`hook`这样的数据结构
    ```js
    const fiber = {
      memoizedState: hook1 => hook2 => hook3..., // 保存该FunctionComponent对应的Hooks链表, 比如有好几个hook，就是 hook1 =》hook2 =》hook3... 
      stateNode: xxx
    };
    // 对应上面Hooks链表里的每个节点
    hook = {
      queue: {
        pending: null // 保存update的queue，和state更新有关，下面会讲
      },
      memoizedState: initialState, // 保存hook对应的state
      next: null // 指向下一个Hook，连接形成链表
    }
    ```
    所以，可以理解为每次`useState`函数取到的值都是Fiber Node里缓存的值，那就不会每次都是返回initialValue
  2. `state`是怎么更新的? 上面只讲了数据结构，那React是怎么更新这个数据结构里的state的呢？
    首先，组件初次渲染（mount）时对应的Fiber Node都没有创建，创建了Fiber Node后上面说的数据结构也需要依次添加到这个Fiber Node中，而update不同，update时已经有这个Fiber Node了，必须使用上一次存在Fiber Node中的值，再更新。
    伪代码表示
    ```js
    if (isMount) {
      let fiber = createFiberNode() // 创建fiber
      element.hooks.map(hook => fiber.memoizedState.add(new Hook(initialState))) // 插入数据结构
    }
    if (isUpdate) {
      fiber = findFiberNode(); // 找到fiber node
      fiber.memoizedState.map(hook => { // 每个hook按序执行
        let state = memoizedState // 取出state
        hook.queue.map(updateTask => { // 执行更新函数
          state = updateTask(state) 
        })
        hook.memoizedState = state; // 保存更新过的state
      })
    }
    ```
    可以看到，Hooks在mount和update阶段对应不同分支，在React真实实现中，**所有的Hooks 在mount和update阶段是被分成两种Dispatcher**
    ```js
    // mount时的Dispatcher
    const HooksDispatcherOnMount: Dispatcher = {
      useCallback: mountCallback,
      useContext: readContext,
      useEffect: mountEffect,
      useImperativeHandle: mountImperativeHandle,
      useLayoutEffect: mountLayoutEffect,
      useMemo: mountMemo,
      useReducer: mountReducer,
      useRef: mountRef,
      useState: mountState,
      // ...省略
    };

    // update时的Dispatcher
    const HooksDispatcherOnUpdate: Dispatcher = {
      useCallback: updateCallback,
      useContext: readContext,
      useEffect: updateEffect,
      useImperativeHandle: updateImperativeHandle,
      useLayoutEffect: updateLayoutEffect,
      useMemo: updateMemo,
      useReducer: updateReducer,
      useRef: updateRef,
      useState: updateState,
      // ...省略
    };
    ```
    明白上面伪代码的逻辑，就可以明白React是怎么初始化函数式组件中state的值。
    同时，setXXX就是一个把更新任务插入到hook.queue中的函数，每次触发组件update，每个hook queue中的action就会被执行，然后得到更新后state的值。

    > 再多解释一个点，Hooks有个默认规矩就是更新函数要叫`setXXX`，但是实际上，这只是一个数组解构的语法，理论上数组解构完全依赖索引 `const array = [64, 80]; const [ a, b ] = array;`，数组也可以理解成key是索引的特殊对象，这里`a`就是64`b`就是80，换成`aaa`或`setAaa`也是一模一样的，所以这只是一种使用层面的约定而不是API的限制，当然推荐遵守这个约定
  
### useReducer
看名字就知道useReducer大概的用法和redux差不多，举个例子
```js
const [schedule, dispatch] = useReducer(
  (state, action) => {
    switch (action.type) {
    case 'init':
      return action.payload;
    case 'change':
      return {
        ...state,
        value: {
          ...state.value,
          visitWeekDay: action.payload.value
        }
      };
    case 'chose':
      return {
        ...state,
        index: action.payload.value
      };
    default:
      return state;
    }
  },
  {
    index: -1,
    value: undefined 
  }
);
```
useReducer完全可以理解成useState的替代品，基本只要遇到需要存储多个state，而且这些state是一个嵌套关系的数据结构选择useReducer就没错了

## useEffect和useLayoutEffect
### side effect
讨论Hooks的前提是在函数式组件中，说到函数式组件，就会涉及到纯函数、函数式编程等概念，这里不多讨论。只说在React中，除了变量计算，返回jsx形式的html模版，其余都是副作用，打日志、远程接口调用、dom操作这些都是，理论上这些操作都需要用useEffect包裹。
### 参数
useEffect包含两个参数，第一个是副作用函数，第二个是dependencies。第二个参数是可选的，传入第二个参数可以使副作用条件执行，如果传空数组则只在组件首次渲染执行。
### 执行时机
useEffect和useLayoutEffect最大的区别就是执行时机。
结论是useEffect执行得非常晚。React在render phase结束之后会标记所有有副作用的节点，然后在commit phase就会遍历effect list。准确地说，对于useEffect，会产生两个数组，一个存储上一次的useEffect产生的销毁函数，另一个才是存储这次的副作用函数。只有第一个销毁函数的数组会在commit phase阶段一开始就被执行。存储副作用的列表只是被标记了一般优先级，这个列表的执行时机是commit phase结束之后，js空闲了才有机会执行，也就是所有的dom改变都已经被呈现在UI界面之后才会执行。
因为useEffect里的副作用生效非常晚，可能导致抖动等渲染bug（就是先完成了渲染dom，useEffect而后执行又渲染了dom），所以官方给了useLayoutEffect这个Hook。这个Hook和useEffect基本没差别，关键差别就是它生成的副作用数组在commit phase阶段就会执行，这样就能避免问题。

## useCallback、useMemo、React.memo
### useCallback、useMemo的区别
太多的文章讲useCallback、useMemo这两个让人费解的API了，总结了各种各样的应用场景，但问题是很多文章理解都是错的。
这两个API，在官方文档有明确说法 `useCallback(fn, deps) 相当于 useMemo(() => fn, deps)`。一句话总结，这俩API useCallback缓存了函数本身，useMemo缓存了函数的执行结果，完全符合上面的公式。
```js
function mountMemo<T>(
  nextCreate: () => T,
  deps: Array<mixed> | void | null,
): T {
  ...
  const nextValue = nextCreate(); // 执行了一下
  ...
  return nextValue;
}

function mountCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  ...
  return callback; // 直接返回callback
}
```

### React.memo
React.memo不属于Hooks，这是React顶层API，这里提一下是因为很多场景下，React.memo被推荐和useCallback、useMemo比较或者配合使用。
明确React.memo的作用，只浅层比较props属性，如果props属性不变则不重新渲染。类似于，函数式组件版本的pureComponent。React.memo还能接收第二个参数，用于自定义prevProps和nextProps的比较规则。
