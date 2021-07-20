# UI组件库
聊聊UI组件库建设的思路

## npm包管理
组件库开发一般需要固定第三方库的版本依赖，npm在这方面很难直接限制，提供的peerDependency只是安装时候的警告，不是强制限制。
目前探索的方法是使用monorepo。
### lerna
前端好用的monorepo工具是lerna。lerna本质上解决的是npm link问题，packages中可以共享第三方依赖。
lerna常用命令
```shell
lerna init 
lerna publish
lerna add 3rdlib --scope=target-package
```
### yarn
yarn的workspace也可以管理依赖，配合lerna使用非常棒。相应的
lerna中的配置项
```json
{
  ...
  "npmClient": "npm" | "yarn",
  ...
}
```

## 版本管理
lerna也能直接解决，lerna的lerna.json中可以配置版本管理模式
```json
{
  ...
  "version": "fixed" | "independent", // 对应 统一和独立字段
  ...
}
```

## 规范化git commit和changelog
### husky
规范化常用的工具是husky，安装husky后，husky可以配置git的执行钩子，一般在钩子中实现lint
### eslint和stylelint
eslint和stylelint可以规范化代码，统一编码风格
### commitizen
使用commitizen可以通过命令行生成一个适配的commit规范，命令行使用git cz即可
### conventional-changelog-cli
这个工具是angular的工具，可以帮我们自定生成changelog

## 开发调试
组件库开发调试除了本身文件夹启动服务以外，还有一种简单的方法是和展示平台统一。
对于react库来讲，storybook是很不错的选择，演示平台可以直接调试代码，配置也很简单
常见的目录结构如下：
[使用storybook](./storybook-dir.png)

## 演示平台
react库推荐使用storybook或者dumi，如果是基于antd二次封装的组件库，推荐dumi（其实如果不想自己折腾，dumi是最完善的方案）
### storybook
storybook的目录结构上面已经放过了，.storybook目录下的main文件可以配置storybook，比如webpack的自定义配置都可以在这里配
```js
const path = require('path');

module.exports = {
  "stories": [
    "../packages/**/*.stories.mdx",
    "../packages/**/*.stories.@(js|jsx|ts|tsx)"
  ],
  "addons": [
    "@storybook/addon-links",
    "@storybook/addon-essentials",
    "@storybook/addon-controls"
  ],
  "webpackFinal": async config => {
    ...
  }
```
## 按需加载
用过antd组件的都知道antd有按需加载的功能，实现方式是用了`babel-plugin-import`
有了这个插件，只需要把组件库每个组件的目录结构都输出成和antd类似结构即可实现按需加载
[babel-plugin-import结构](./babel-plugin-import-dir.png)
如何实现这个结构，是打包要考虑的问题
## 打包
### 产物形式
如果以antd为例，输出3种形式，es、commonJS和UMD，相应的package.json中也需要配置`main`、`browser`、 `browser`告诉用户对应的入口。
但是真的组件库一定需要这三种形式吗？当然不是，具体的模式应该按照自己的需求而定。举例，如果组件库规模不大，且只需兼容内部客户端使用，完全可以只有es module形式的一种产物。
### 打包方式
做一些比对
#### webpack
结论：不是最好的方式。优点：可以打包几乎所有类型的资源。缺点：不适合输出ES Module格#式
#### rollup
结论：组件库不错的打包方式，但是不适合单用。优点：适合输出ES Module。缺点：样式、图片其它资源可以处理，但是很难分割，除非执行多次打包
#### tsc
结论：不错的打包方式。优点：如果是ts代码可以直接使用tsc编译。缺点：也不适合处理样式、图片等资源
### babel
结论：类似tsc。
#### gulp
结论：可以完成所有打包任务，但是一般只用来处理部分格式的文件。
最终的方案：
  可以使用rollup处理js文件，或者babel直接处理也可以，ts的声明文件使用tsc生成。样式使用postcss处理，其余格式的资源可以不处理或者也可以使用相应的插件处理（比如图片转base64），这部分流程是一些任务处理，可以使用gulp完成。
#### dumi
前面已经提过了dumi，dumi提供了rollup和babel两种打包方式，相关资源也是通过gulp和gulp插件处理，整体思路和前文的方案基本一致。

## 测试
这部分内容后续再补充
### 单元测试
jest
### e2e测试
### 集成测试
puppeteer


## CI/CD
CI/CD比较重要的是工具选择。比如在github平台，github action即可解决问题。常用的解决方案还有第三方的

参考：
  [lerna中文网](https://www.lernajs.cn/)
