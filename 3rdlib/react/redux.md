# redux原理

## 官方示例用法
```javascript
import { createStore } from 'redux'

/**
 * This is a reducer, a pure function with (state, action) => state signature.
 * It describes how an action transforms the state into the next state.
 *
 * The shape of the state is up to you: it can be a primitive, an array, an object,
 * or even an Immutable.js data structure. The only important part is that you should
 * not mutate the state object, but return a new object if the state changes.
 *
 * In this example, we use a `switch` statement and strings, but you can use a helper that
 * follows a different convention (such as function maps) if it makes sense for your
 * project.
 */
function counter(state = 0, action) {
  switch (action.type) {
    case 'INCREMENT':
      return state + 1
    case 'DECREMENT':
      return state - 1
    default:
      return state
  }
}

// Create a Redux store holding the state of your app.
// Its API is { subscribe, dispatch, getState }.
let store = createStore(counter);

// You can use subscribe() to update the UI in response to state changes.
// Normally you'd use a view binding library (e.g. React Redux) rather than subscribe() directly.
// However it can also be handy to persist the current state in the localStorage.
store.subscribe(() => console.log(store.getState()))

// The only way to mutate the internal state is to dispatch an action.
// The actions can be serialized, logged or stored and later replayed.
store.dispatch({ type: 'INCREMENT' })
// 1
store.dispatch({ type: 'INCREMENT' })
```
总结一下，`redux`的核心`API`非常简单
`redux`顶级暴露的方法，也就是我们要实现的方法
- `createStore(reducer, [preloadedState], [enhancer])`
- `combineReducers(reducers)`
- `applyMiddleware(...middlewares)`
- `bindActionCreators(actionCreators, dispatch)`
- `compose(...functions)`

其中`store`的`API`
- `getState()`
- `dispatch(action)`
- `subscribe(listener)`
- `getReducer()`
- `replaceReducer(nextReducer)`

## function createStore()
`functuon createStore: (reducer, [preloadedState], [enhancer]) => store`
```javascript
  const createStore = (reducer, preloadedState, enhancer) => {
    return {
      getState: () => {},
      dispatch: action => {},
      subscribe: listener => {},
      getReducer: () => {},
      replaceReducer: nextReducer => {}
    };
  };  
```
### store.getState()
```javascript
  const createStore = (reducer, preloadedState, enhancer) => {
    let currentState = preloadedState;
    return {
      getState: () => currentState
    };
  };  
```
### store.subscribe(listener)
从名字易得是实现一个订阅者模式。
提一下`unsubscribe`的方法如下,所以这个函数的返回值是一个`unsubscribe`函数，`store`本身也未提供`unsubscribe`方法
```javascript
const unsubscribeHandler = store.subscribe(handler)
unsubscribeHandler()
```
```javascript
const createStore = (reducer, preloadedState, enhancer) => {
    ...
    let listeners = [];
    return {
      ...,
      subscribe: listener => {
        listeners.push(listener);
        return () => {
          const index = listeners.indexOf(listener)
          listeners.splice(index, 1)
        }
      }
    };
  };
```
>上述存在问题，`listener` 被触发期间如果`unsubscribe`则影响`listener`的顺序
假如现在有3个listener [A,B,C], 遍历执行，当执行到B的时候(此时下标为1)，B 的内部触发了unsubscribe 取消订阅者B，导致变成了[A,C],而此时下标再次变为2的时候，原本应该是C的下标此时变成了1，导致跳过C未执行。快照的作用是深拷贝当前listener，在深拷贝的listener上做事件subscribe与unSubscribe。不影响当前执行队列

```javascript
const createStore = (reducer, preloadedState, enhancer) => {
  let currentListeners = [];
  let nextListeners = currentListeners;
  subscribe: listener => {
    if (nextListeners === currentListeners) {
      nextListeners = currentListeners.slice()
    }
    nextListeners.push(listener);
    return () => {
      if (nextListeners === currentListeners) {
        nextListeners = currentListeners.slice()
      }
      const index = nextListeners.indexOf(listener)
      nextListeners.splice(index, 1);
      currentListeners = null;
    }
  }
}  
```

# store.dispatch
```javascript
const createStore = (reducer, preloadedState, enhancer) => {
  ...
  let currentState = preloadedState;
  return () => {
    dispatch: currentState = reducer(currentState, action)
  }
}  
```

合并一下`store`的代码
```javascript
const createStore = (reducer, preloadedState, enhancer) => {
    let currentState = preloadedState;
    let currentListeners = [];
    let nextListeners = currentListeners;
    return {
      getState: () => currentState,
      subscribe: listener => {
        if (nextListeners === currentListeners) {
          nextListeners = currentListeners.slice()
        }
        nextListeners.push(listener);
        return () => {
          if (nextListeners === currentListeners) {
            nextListeners = currentListeners.slice()
          }
          const index = nextListeners.indexOf(listener)
          nextListeners.splice(index, 1);
          currentListeners = null;
        }
      },
      dispatch: currentState = reducer(currentState, action)
    };
  };
```

# react-redux
首先在最外层要提供一个`Provider`组件
```javascript
import React from 'react'
import PropTypes from 'prop-types'
export class Provider extends React.Component {  
  // 需要声明静态属性childContextTypes来指定context对象的属性,是context的固定写法  
  static childContextTypes = {    
    store: PropTypes.object  
  } 

  // 实现getChildContext方法,返回context对象,也是固定写法  
  getChildContext() {    
    return { store: this.store }  
  }  

  constructor(props, context) {    
    super(props, context)    
    this.store = props.store  
  }  

  // 渲染被Provider包裹的组件  
  render() {    
    return this.props.children  
  }
}
```
`connect()`方法的`API`如下：
```javascript
connect(mapStateToProps, mapDispatchToProps)(App)
```
```javascript
//react-redux.js
import React from 'react'
import PropTypes from 'prop-types'
export class Provider extends React.Component {  
  // 需要声明静态属性childContextTypes来指定context对象的属性,是context的固定写法  
  static childContextTypes = {    
    store: PropTypes.object  
  }  

  // 实现getChildContext方法,返回context对象,也是固定写法  
  getChildContext() {    
    return { store: this.store }  
  }  

  constructor(props, context) {    
    super(props, context)    
    this.store = props.store  
  }  

  // 渲染被Provider包裹的组件  
  render() {    
    return this.props.children  
  }
}

export function connect(mapStateToProps, mapDispatchToProps) {    
  return function(Component) {      
    class Connect extends React.Component {        
      componentDidMount() {          //从context获取store并订阅更新          
        this.context.store.subscribe(this.handleStoreChange.bind(this));        
      }        
      handleStoreChange() {          
        // 触发更新          
        // 触发的方法有多种,这里为了简洁起见,直接forceUpdate强制更新,读者也可以通过setState来触发子组件更新          
        this.forceUpdate()        
      }        
      render() {          
        return (            
          <Component              
            // 传入该组件的props,需要由connect这个高阶组件原样传回原组件              
            { ...this.props }              
            // 根据mapStateToProps把state挂到this.props上              
            { ...mapStateToProps(this.context.store.getState()) }               
            // 根据mapDispatchToProps把dispatch(action)挂到this.props上              
            { ...mapDispatchToProps(this.context.store.dispatch) }             
          />          
        )        
      }      
    }      

    //接收context的固定写法      
    Connect.contextTypes = {        
      store: PropTypes.object      
    }      
    return Connect    
  }
}  
```
