
thread-loader

Speed Measure Plugin: 显示打包任务执行时间

## 页面优化的几个点
1. 单个页面的代码如果有多模块，怎么保证合并
2. 首页的资源怎么优先加载，不需要的异步加载

## webpack基础配置项
### webpack是否必须有配置文件
webpack可以没有配置文件，理论上webpack默认会以src/index.js为入口，然后进行打包。但是，这种情况下webpack只能处理js文件，同时没法使用插件。
### webpack配置文件
讨论具体配置项之前，先得搞明白webpack的配置文件放在哪儿。
1. 默认选项，webpack会读取和package.json同级目录下的webpack.config.js。
2. 读取自定义配置项，在npm script里注册config文件地址，webpack即会读取自定义配置项
```js
"scripts": {
  "build": "webpack --config prod.config.js"
}
```
3. 多种情形配置。最常见的是区分本地调试和线上打包配置。基本使用方式是webpack的配置文件接受一个函数（可以返回Promise解决异步场景），只需要在这个函数中区分环境即可。
```js

```
### 具体配置项介绍
- entry
- output
这两个配置项是必须的，作用很明显，用于配置打包入口和输出。
- mode：mode只有三个选项(development | production | none)，也是必须的，作用是告诉webpack这是什么环境的打包，方便webpack使用针对不同环境的优化打包逻辑，可以理解为mode属性提供的选项就是官方提供的`optimization`配置合集。
- module：必须配置，webpack处理各种类型的文件使用什么规则（一般指loader）在此处配置。
  ```js
  module {
    rules: [{ // 需要加一层rules
      test: /\.jsx?$/,
      include: [
        path.resolve(__dirname, "app")
      ],
      exclude: [
        path.resolve(__dirname, "app/demo-files")
      ],
      // 这里是匹配条件，每个选项都接收一个正则表达式或字符串
      // test 和 include 具有相同的作用，都是必须匹配选项
      // exclude 是必不匹配选项（优先于 test 和 include）
      // 最佳实践：
      // - 只在 test 和 文件名匹配 中使用正则表达式
      // - 在 include 和 exclude 中使用绝对路径数组
      // - 尽量避免 exclude，更倾向于使用 include
      issuer: { test, include, exclude },
      // issuer 条件（导入源）
      enforce: "pre",
      enforce: "post",
      // 标识应用这些规则，即使规则覆盖（高级选项）
      loader: "babel-loader",
        // 应该应用的 loader，它相对上下文解析
        // 为了更清晰，`-loader` 后缀在 webpack 2 中不再是可选的
        // 查看 webpack 1 升级指南。
      options: {
        presets: ["es2015"]
      }
    }]
  }
  ```
- plugin： 所有插件在此处配置
- resolve：非必需配置，但是一般都会配，作用是让webpack找到模块路径。比较常规的配置是`extensions`统一注册后缀名，import语句就不需要带上文件名后缀，`modules`可以让webpack直接从对应路径下查找加载的模块，`alias`可以配置模块路径的`aliasPath`，`mainFields`解决第三方包如果有多个输出版本代码里加载哪个路径下的版本。
- optimization：非必需配置，只需要配置mode，webpack默认就提供配置策略，但是也可以自己手动修改

**特别说明，webpack有多个版本，到目前为止有v1-v5，一般当下项目中多是v3以上，老项目可能v4居多，新项目新脚手架匹配的多是v5，在使用过程中一定要注意loader、plugin和webpack版本的关系，很多报错是由于大版本不匹配导致的。**

## 真正掌握webpack
知道上面的知识就足够使用webpack了，但是回答完下面这些问题才能自如使用webpack。
### 配置文件
1. 怎么区分开发模式和打包模式？
  webpack官方文档上就给出了一种解决方案，引入`webpack-merge`插件，将配置文件拆分成三份
  ```
  |-webpack.common.js // 所有公共部分配置
  |-webpack.dev.config.js // 开发模式下的配置
  |-webpack.prod.config.js // 打包模式下的配置
  ```
  ```json
  {
    "script": {
      "dev": "webpack-dev-server --config='path/to/webpack.dev.config.js'",
      "build": "webpack --config='path/to/webpack.prod.config.js'",
    }
  }
  ```
  其实理解了这种方案，也很好看出本质上只不过是执行npm script的时候读取了不同的config配置，`weback-merge`插件最重要的作用是提取公共部分代码，遵循`DRY（donot repeat yourself）`原则。
  如果很简单的项目需要自己手动配置，直接在配置文件代码里用三目运算符区分环境变量即可。
  如果不想用`weback-merge`插件，自己按照js语法处理合并对象也都是一样的。
  最终目标只要执行命令时能读取到合适的配置文件即可。
### loader相关
1. 各种资源用什么loader？
  loader本身是用来处理各种资源的，对应不同的资源有不同的loader。
  按照前端三要素html+css+js来区分：
  js：这里的js是最后执行时候的js，源码层面可能有.ts、.vue等等各种文件。最常见的loader包括以下：
    - babel-loader: 虽然webpack本身就能识别js，但是因为js语法上遵循不同的es标准，多半需要使用babel-loader编译
    - ts-loader: 处理ts的loader
    - vue-loader: vue框架下的SFC使用.vue文件，所以需要vue-loader处理。严格来说vue SFC并不仅仅是js，它还包括html片段和css片段，所以vue-loader本身还依赖其他的vue-style-loader这些loader
  css: webpack默认只能处理js，css文件需要相应的loader来处理
    - css-loader: 处理css文件
    - style-loader: 这也几乎是必须的一个loader，css-loader作用是把css文件转换成webpack能识别的模块，但是css文件并不能在js中生效，需要有style标签插入html，style-loader就是这个作用
    - mini-css-extract-plugin.loader: 可以看到mini-css-extract-plugin是一个plugin，这个plugin提供了一个loader，使用这个插件可以将css的chunk单独分离出来替代style-loader。实际使用中考虑到打包速度，开发模式使用css-loader+style-loader，打包模式使用css-loader+mini-css-extract-plugin.loader分离css文件。
    - less-loader： 处理less文件，因为是css预处理，所以需要作为第一个loader使用
    - sass-loader：和less-loader一样，用于处理sass文件
    - postcss-loader：
  html: 很多情况下html是不需要loader的，原因不是因为webpack能处理html，目前使用react、vue等前端框架的情况下html入口只需要提供一个root标签，多数情况直接使用htmlWebpackPlugin生成html。
    - html-loader: 处理html文件，多数场景下是需要一个html模版，本地已经编写html，然后通过一系列操作直接返回这个html，这时候需要html-loader
  其它资源：
    - file-loader: 处理各种格式的图片等静态资源
    - url-loader: 作用和file-loader一样，但是处理结果是内联样式，适合处理小图片、字体文件等
    需要说明的是，webpack v5开始，file-loader和url-loader就不需要了，webpack配置文件中只需要配置不同的type即可达到原来上述loader的效果，上述loader也将被废弃。
    ```js
    exports.module = {
      ...
      module: {
        rules: [{
          test: /\.(png|jpg|webp|git)\/i,
          type: 'javascript-auto' | 'asset/resource' | 'asset/inline' | 'asset/source'
        }]
      }
    }
    ```
2. loader的执行顺序
  loader执行顺序是从右到左，例如`loader: ["style-loader", "css-loader"]`。（*曾经看到过有人提问为“什么laoder的执行顺序不是从左到右？”。这只是一个规定，没什么值得讨论的，如果从左到右了又可以问“为什么不是从右到左？”，但是在计算机里有两个例子，一种是pipe模式`first()|second()|third()`，一种是函数式编程`f(g(h(x)))`，显然这两种情况一种就是从左到右，另一种是从右到左。js里也有reduce和reduceRight。知道loader执行顺序很简单，某些看似无聊的问题思考一下各种可能的实现方式，并借鉴一些经典的例子能得到更多。*）
3. loader的工作原理
### plugin相关
1. 常用的plugin
2. pulgin的工作原理
### 代码分割
1. 同步分割
  webpack因为打包后的包体积可能比较大，不利于页面加载，所以需要代码分割。可以配置optimization.chunkSplit和runtimeChunk解决
  ```js
  optimization: {
    runtimeChunk: true, // webpack维护各种chunk id和chunk之间有一个映射关系表，这个关系表默认在入口文件打包的file下，runtimeChunk选项作用就是把这个映射表emmit，不至于只是映射关系的变化就导致入口文件变化缓存失效
    splitChunks: {
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/, // 分割哪部分代码
          name: 'vandors',
          priority: -30, // 分割规则的优先级
          chunks: "all", // 可以配置分割异步代码还是所有代码
          reuseExistingChunk: true, // 是否复用其它模块里的重复部分，true表示复用其它模块，false会把共用部分抽离不复用
          minSize: 20000, // 压缩前最小体积达到多少才开始分割
          maxSize: 50000, // chunk最大体积，大于这个体积则把chunk再拆分
          minChunks: 2 // 至少被引用多少次才单独分割
        }
      }
    }
  }
  ```
  (早期版本有个commonsChunk的插件，webpack v4后这个插件可以被上述配置项替代)
2. 异步分割
  webpack支持异步分割且无需额外配置。对于dynamic import语句，webpack自动会把引入模块单独放入一个chunk。
3. 按需分割
  上面的代码同步分割中已经体现了部分被重复引用的代码可以被单独打包。如果更个性化，需要把特定文件单独打包也是可行的，一般常见的需求是把`react`、`vue`、`lodash`、`moment`等这些第三方库单独抽离打包。实际操作中，有两种思路：1. 直接配置`external`，忽略这些文件，然后使用中可以直接使用线上CDN资源。2. 使用webpack自带的dllPlugin/dllReferencePlugin插件（具体介绍看上方plugin部分）。
### 热更新
1. HMR是什么？怎么实现的？
### tree shaking
1. webpack怎么tree shaking？
  webpack4打包过程中的tree shaking开启是很简单的，只需要在package.json中添加sideEffects字段，webpack就会在编译过程中使用tree shaking。webpack5更简单，默认tree shaking就是开启的，意味着什么都不做，代码就会被tree shaking。
  当然tree shaking并没有这么简单，tree shaking的核心逻辑和js的模块化方式有很大关系。对比两段代码
  CommonJS
  ```js
  const a = require('a');
  const b = require('b');

  if (someCondition) {
    a();
  } else {
    b();
  }
  ```
  ES Module
  ```js
  import { a } from 'a';
  import { b } from 'b';

  if (someCondition) {
    a();
  } else {
    b();
  }
  ```
  CommonJS标准下是动态导入，所以根据条件不同，`a`和`b`是有可能不加载的.ES Module是静态加载，无论如何在头部声明完两句import语句后，就会导入`a`、`b`模块。
  tree shaking和上面的例子关系很大，webpack4中只会对ES Module模块进行tree shaking，webpack5对tree shaking功能做了大升级，部分CommonJS模块也可以tree shaking。
### 打包css
1. 如何处理css资源
2. css的兼容性问题怎么解决？
3. css的代码分割
### webpack5的新特性
1. 模块联邦
2. 其它特性更新
  1. 使用资源模块type：asset/resource、asset/inline、asset/source替代raw-loader、file-loader、url-loader
  2. tree-shaking加强
  3. webpack-dev-server目前还需要使用@next版本
### webpack性能优化
1. 怎么查看webpack的打包结果？
2. 优化可以从哪几个方面入手？
### webpack的基本原理
1. webpack的基本原理