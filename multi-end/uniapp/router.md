## uniapp路由

### uniapp自带路由
#### 一般用法
uniapp自带的路由系统比较简单，基本和微信小程序一致，所有页面都注册在pages.json文件下，path直接对应pages文件夹下的路径，无需配置文件和路径之间的映射关系。
缺点也很明显，这种路径配置模式因为开放的API很简单，所以提供的能力也很简单，最常见的需求比如路由守卫是没法实现的，解决方法就是引入插件。
#### 路由传参
简单的路由传参直接在url链接后面加参数即可。
复杂的理由传参，在2.8.9+版本后，支持使用uni.eventChannel传递参数。这个API和微信小程序提供的API风格一致。
```js
// 2.8.9+ 支持
uni.navigateTo({
  url: 'pages/test?id=1',
  events: {
    // 为指定事件添加一个监听器，获取被打开页面传送到当前页面的数据
    acceptDataFromOpenedPage: function(data) {
      console.log(data)
    },
    someEvent: function(data) {
      console.log(data)
    }
    ...
  },
  success: function(res) {
    // 通过eventChannel向被打开页面传送数据
    res.eventChannel.emit('acceptDataFromOpenerPage', { data: 'test' })
  }
})

// uni.navigateTo 目标页面 pages/test.vue
onLoad: function(option) {
  console.log(option.query)
  const eventChannel = this.getOpenerEventChannel()
  eventChannel.emit('acceptDataFromOpenedPage', {data: 'test'});
  eventChannel.emit('someEvent', {data: 'test'});
  // 监听acceptDataFromOpenerPage事件，获取上一页面通过eventChannel传送到当前页面的数据
  eventChannel.on('acceptDataFromOpenerPage', function(data) {
    console.log(data)
  })
}
```


### uni-simple-router
- 使用方式
  这里只说兼容模式。如果项目只针对h5，可以有h5模式特有的配置方法，使用起来和vue-router基本一致。
  这里假设项目是用vue-cli搭建的dcloudio/uni-preset-vue默认模版
  1. 安装
    通过npm方式：`npm install uni-simple-router`
  2. 必要的修改
    main.js文件，调用自定义的router，添加针对h5的条件编译
    ```js 
    ...
    import { router, RouterMount } from './router.js'
    ...
    Vue.use(router)
    ...
    // #ifdef H5
    RouterMount(app, router, '#app')
    // #endif

    // #ifndef H5
    app.$mount()
    // #endif
    ```
    创建router.js文件，用法类似于vue-router，定义router。
    ```js
    import {
      RouterMount,
      createRouter,
      runtimeQuit
    } from 'uni-simple-router';

    const router = createRouter({
      ...
      routes: [
        ...ROUTES, // 这里可以复制pages.json中的配置
        {
          path: '*',
          redirect:(to)=>{
            return {name:'404'}
          }
        },
      ]
    });

    export {
      router,
      RouterMount
    }
    ```
    这里`createRouter`方法的参数可以参考官方文档，需要特别说明routes项。首先，routes必须把pages.json中的页面配置再重新配置一遍才会使路由生效（原理是pages.json下的路由是所有路由，此处配置的是main.js中生效的路由）。考虑到这两份路由是一样的，所以作者提供了一种使用webpack.DefinePlugin的方法读取pages.json中的路由，然后以全局变量ROUTES的形式在项目中直接使用路由变量ROUTES。
    使用这种方法需要这样设置，
    `npm install uni-read-pages`
    然后配置vue.config.js文件，定义全局变量
    ```js
    const TransformPages = require('uni-read-pages')
    const {webpack} = new TransformPages()
    module.exports = {
      configureWebpack: {
        plugins: [
          new webpack.DefinePlugin({
            ROUTES: webpack.DefinePlugin.runtimeValue(() => {
              const tfPages = new TransformPages({
                includes: ['path', 'name', 'aliasPath']
              });
              return JSON.stringify(tfPages.routes)
            }, true )
          })
        ]
      }
    }
    ```
  3. 注意事项
    - aliasPath
    uniapp默认的path配置不能省略`pages/xxx`这些公共路径，使用`uni-simple-router`是可以通过aliasPath重写的。使用过程中，发现aliasPath不是简单的提供一个alias，而是覆盖path，使用后原有的path会失效。
  4. 路由传参
     如果使用路由插件，因为路由插件的封装，建议使用`query`传递参数，不要再使用eventChannel
     