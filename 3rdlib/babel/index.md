本篇讨论babel相关话题

*本篇只谈论`babel@7.x`相关内容* 

老规矩：[babel的一切请查阅babel官网](https://babeljs.io/)


# babel的重要概念
## 包结构
`babel@7.x`项目的包管理方式遵从`monorepo`模式 //todo here what is monorepo, intruduce in build an UI library

## @babel/core
`@babel/core`是整个项目的核心，`babel`的编译函数就在这个核心包内，理论上只有一个`@babel/core`就可以使用`babel`编译了
```javascript
const { transform } = require("@babel/core");
transform(
  "code();", // raw code
  {}, // babel config
  function(err, result) { // transform callback
    console.log(
      result.code,
      result.map,
      result.ast
    );
});
```

## @babel/cli
`@babel/cli`使得`babel`命令在终端可用

## 配置文件
上文讲到的`@babel/core`中的转换函数接收一个配置参数，这个参数能接收的字段如下：
```
Primary options
Config Loading options
Plugin and Preset configuration
Config Merging options
Source Map options
Misc options
Code Generator options
AMD / UMD / SystemJS options
Option concepts
```
*文档中有标注，如果直接传参数给`@babel/cli`，需要用kebab-case（破折号连接）处理键值*
当然了，实际上在很多开发场景下，用户并不是直接调用`@babel/cli`，我们用到的配置文件都写在`babel.config.js`文件中

## Plugins
`Plugins`表示`babel`转换中用到的小插件，比如`@babel/plugin-transform-arrow-functions`就是转换箭头函数的插件。
光有这个插件还不行，还要让这个插件生效，这就涉及到上面讲的`@babel/cli`和`@babel/core`了。
串联一下逻辑，上面讲了`@babel/cli`让`babel`命令在终端环境中可用，`@babel/core`中有`babel`真正的转换方法，同时转换方法是可以接收配置参数的，那串联起这两个包要做的不外乎就是把命令行参数转换为配置参数，再传给`babel`转换方法。
例子：
```shell
./node_modules/.bin/babel src --out-dir lib --plugins=@babel/plugin-transform-arrow-functions // 注意后面传入了plugin参数
```
显然，命令行中的`--plugins`被传入到了转换函数的配置参数中

## @babel/env
`preset`是配置语法糖，避免的场景是手动配置一个又一个`plugin`，这里有这些重要知识点：
 这个preset是怎么生效的，答案是browserlist。browserlist返回一个浏览器最低版本支持语法的列表，babel就可以依照这个列表配合用户自己设置的targets，替换需要转码的语法（syntax）。需要说明的是useBuiltIns分为`false ｜ entry ｜ usage`，这里要配合core-js使用，因为babel默认只会引入不支持的关键字语法，不能直接替换Promise.allSettled这样的方法(api)，core-js能引入polyfill。`entry/usage`对应的是不同的引入方式，一般usage更加合适，entry太暴力。

## @babel/plugin-transform-runtime
 @babel/env非常好用，但是也有缺陷，主要问题就是重复引入polyfill，最关键是会替换Array的原型链污染全局。所以如果做第三方库，最好使用@babel/plugin-transform-runtime插件，可以统一引入@babel/helper，配合core-js3使用是完美的。

# babel的工作原理
babel-cli开始读取我们的参数(源文件test1.js、输出文件test1.babel.js、配置文件.babelrc)
babel-core根据babel-cli的参数开始编译
Babel Parser 把我们传入的源码解析成ast对象
Babel Traverse（遍历）模块维护了整棵树的状态，并且负责替换、移除和添加节点(也就是结合我们传入的插件把es6转换成es5的一个过程)
Babel Generator模块是 Babel 的代码生成器，它读取AST并将其转换为代码和源码映射（sourcemaps）。

参考：
  - [babel源码解析一](https://blog.csdn.net/vv_bug/article/details/103823257)
