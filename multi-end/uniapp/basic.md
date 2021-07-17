## 开发uniapp过程中遇到的问题合集
### 条件编译
因为uniapp是面向多端的，90%的情况可以用一套代码，还有10%的情况需要兼容各端特性，所以出现了条件编译。
条件编译的语法参考c语言，采用
```
// #ifdef %PLATFORM%
code here
// #endif
// #ifndef %PLATFORM%
code here
// #endif
```

uniapp中几乎所有代码都可以走条件编译，逻辑代码、样式、.json文件中采用的都是注释语法
- js部分
  ```
  // #ifdef %PLATFORM%
  code here
  // #endif
  ```
- 样式部分
  ```
  /*  #ifdef  %PLATFORM%  */
  平台特有样式
  /*  #endif  */
  ```
- 页面部分
  ```
  <!--  #ifdef  %PLATFORM% -->
  平台特有的组件
  <!--  #endif -->
  ```
- pages.json
  ```
  // #ifdef %PLATFORM%
  {
    pages here
  }
  // #endif
  ```

不在单行/多行代码层面，整体代码文件也可以条件编译
例如，static目录下可以新建以%PLATFORM%命名的文件夹，里面的资源文件只会在特定平台被编译

如果要区分页面文件，可以直接新建platforms的目录，然后在下面建立%PLATFORM%命名的文件夹，这里面的所有`.vue`代码都会在特定平台编译

参考：
  [uniapp官网-条件编译](https://uniapp.dcloud.io/platform)