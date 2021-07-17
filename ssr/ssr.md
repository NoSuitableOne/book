# SSR的动机
前端技术演进是个轮回。没有js，服务端直接查数据库，利用自己的模版引擎组装模版，时至今日大部分后端框架的自带功能都有能力完成这些工作 -> js登场，出现专门解决web界面问题的细分领域-前端，初始阶段前端使用3大模块（HTML+CSS+JS），渲染数据的部分专门由前端完成，前后端通信一般使用AJAX -> 前端逻辑日益复杂，出现一些js框架解决底层的渲染问题，前端领域开始注重数据逻辑处理 -> 又发现依赖js框架的页面不具有SEO优势，首次加载也开始缓慢，所以出现了SSR。
理解一下上面的发展历程，前端和任何技术一样都是在轮回中发展前行，每一次轮回又比之前站在更高的历史站位上。

SSR就是在服务端将已经渲染好的页面返回，而非返回一个空节点在客户端利用js逻辑开始加载页面。
几个需要区分辨识的问题：
- SSR是否和后端框架的模版引擎一样？
  不一样。首先架构就不一样，可以理解一下上面的话，web开发发展到前后端分离的阶段，前后端就已经分离了，上面其实只说了前端部分的演变史，后端部分也在一直发展，比如微服务化等等，后端在前后端分离之后就已经专注于数据吞吐，所以即使使用了服务端渲染也是在前端层扩展了能力，前后端分离后定义的分界线没有出现变化。其次，旧版的模版引擎渲染是整个页面为单位吞吐，现代网站都是局部刷新，SSR也是局部刷新。
- SSR为什么能实现？
  虚拟dom。核心就是虚拟dom，可以理解一下，服务器是没有window/document等browser API的，服务器能返回html模版处理的都是js逻辑，所以一切dom操作都是虚拟dom操作。
- SSR的意义？
  1. SEO。针对vue、react这种框架，SSR对SEO的提升是确定的，毋庸置疑。
  2. 首页白屏问题。SSR确实能减少首页白屏时间，但是绝不是解决了白屏问题的全部，也不是解决白屏问题的唯一途径。从解决问题看，SSR能保证节点的渲染，但不能解决页面的假死，客户端加载js和执行js绑定事件等逻辑依然需要时间，假死问题依旧存在。从解决途径看，preRender这些手段也是很不错的方法。

# SSR的基本解决方案（以react为例）
## render
首先，前文说了虚拟dom扮演的角色。以react为例，react架构设计中，react和react-dom就是分开的，相应的还有react-native、react-dom/server等。
需要server端返回编译过的html模版，就需要调用和客户端`reactDOM.render`相对应的API。
react-dom/server下提供了`renderToString/renderToNodeStream/renderToStaticMarkup/renderToStaticNodeStream`等API。
相应的，reactDOM下也需要使用ReactDOM.hydrate替换reactDOM.render复用DOM节点。
## 路由同构
react路由使用的是react-router，如果使用react-router的browser模式，显然window.history下的API是无法在服务端使用的，所以需要使用
```js
<StaticRouter location={req.path}>
  {Routes}
</StaticRouter>
```
当然，这里还有一个更复杂的问题，如果需要像客户端一样按路由进行代码分割，react提供的lazyLoad是无效的，所以需要一些类似`loadable-components`这样的插件配合webpack代码分割。
## redux同构
redux的使用方法基本是没有区别的，唯一的问题是store共享。看代码
```js
const store = createStore(...);
...
<Provider store={store}>
  <App />
</Provider>,
```
问题是，如果代码在客户端执行一遍，没有问题，但是到服务端也执行一遍，那么每次调用createStore产生的实例其实是共享的，这样不同用户之间的数据会出现混乱。
解决方案就是改造createStore方法，暴露返回createStore的方法，那么每次都会新建一个store，保证数据独立。
```js
export default () => {
  return createStore(rootReducer,applyMiddleware(thunk))
};
```
第二个问题是，服务端的生命周期里是只有初次render的，没有update，所以首次render的时候所有props里的数据都需要准备好，包括redux（redux当然也是通过props影响组件）。所以这需要在路由中给每个组件添加一个loadData的属性，作用就是执行store.dispatch()，将redux中的数据提前处理好。同时，这里需要使用路由的匹配功能，匹配所有组件，然后最后执行Promise.all返回所有结果。
```js
<Switch>
    <Route path="/a" exact component={A} loadData={()=>A.loadData()}/>
    <Route path="/b" exact component={B} loadData={()=>B.loadData()} />
</Switch>
```
除此之外，执行完的结果返回客户端需告知客户端同步最新的redux store数据，解决方案是将store.dispatch执行后的数据多插入一个`<script>Window.__store__ = {store.getData()}</script>`这样的标签。客户端在设置store.initialState的时候首先去Window对象下获取一下。
## 样式同构
在SSR中，样式问题是一个可以简单，也可以很复杂的问题，完全取决于使用的样式方案。首先明确，服务端是没有css的概念的，也不能进行dom操作，所以一切传统的样式方案，包括样式预处理方案，都会很复杂，相反如果直接使用css-in-js这样的解决方案，css就成了js，无需任何额外处理。
说一下较复杂的传统样式方案。考虑代码还是用webpack打包，css都需要`css-loader`/`style-loader`处理。其中`style-loader`肯定是不能用了，因为`style-loader`本质上是一个dom操作，将script标签插入dom节点。替代方案是`isomorphic-style-loader`。
组件需要使用`isomorphic-style-loader/withStyles`包裹成高阶组件`withStyles(style)(Component)`
服务端需要使用`'isomorphic-style-loader/StyleContext';`像redux-react一样把所有样式添加到Context中，同时封装每个组件获取的方法style._getCss可以获取到相应的css添加到`css store`中
客户端就是消费传入的`css store`中的css

# SSR框架（以next为例）
实话说，上面的玩法适合大家理解SSR原理，但是不适合真正开发。真实开发中，还是使用框架最方便，一般常用的就是react生态圈的next和vue生态圈的nuxt。这里只介绍next。
首先，上面的所有内容包括路由、路由代码分割、css都不需要任何配置，一切都是内置的（这就是用框架的理由）。稍有不同的redux，使用redux，需要自行安装一个`next-redux-wrapper`，或者也可以参考官方提供的[集成redux方案](https://www.nextjs.cn/docs/faq)


参考资料：
[彻底理解服务端渲染](https://github.com/yacan8/blog/issues/30)
[从零开始，揭秘React服务端渲染核心技术](https://segmentfault.com/a/1190000019916830)
[React 中同构（SSR）原理脉络梳理](https://zhuanlan.zhihu.com/p/47044039)
