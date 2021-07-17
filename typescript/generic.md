## 泛型
泛型是在各种编程语言中很常见的概念，但是很多学习typescript的同学都是从javascript入坑，javascript里是没有泛型概念的，这也成为了掌握typescript的一大难点。

### 泛型必须知道的知识
1. 泛型的语法
  刚刚接触泛型的初学者，最难理解的其实是语法。比如下面这行代码就完全是用泛型运算定义的一个type。
  ```ts
  export declare type EachTestFn<EachCallback extends TestCallback> = (...args: Array<any>) => ReturnType<EachCallback>;
  ```
  简单解释一下，需要理解这行代码先要理解语法，比如`<>`里代表的是参数类型，这里的`EachCallback`就可以在后面使用。`extends`、`RetuenType`这些是ts自带的关键字，其中`extends`用于泛型的类型限制，`RetuenType`则是ts泛型工具。
  这里不讨论这些基础语法，可以参考这些资料理解基础语法[ts中文网](https://www.tslang.cn/docs/handbook/generics.html)、[你不知道的 TypeScript 泛型（万字长文，建议收藏）](https://lucifer.ren/blog/2020/06/16/ts-generics/)

2. ts关键字
  ts关键字不属于泛型的内容，但是泛型其实是用来方便用户做类型计算的，所以必须要理解ts中的关键字才能理解泛型运算。
  - extends
    extends最常出现在三元运算中`T extends U ? X : Y`，表示集合T被集合U包含
  - typeof
    这个关键字非常常用，因为ts是进行变量运算的，比如有变量`const a`/`function fn`，这里的`a`、`fn`对于ts都没有意义，因为ts不关心变量只关心变量的类型，所以做类型运算的时候`typeof a`/`typeof fn`即可得到类型
  - in
    in相当于js中的遍历操作，遍历得到对象中的键值
  - keyof
    keyof也比较常见，可以直接获得对象中的键值
  - infer
    infer算一个比较难理解的关键字，简单讲，infer可以理解成`var/let/cont`，比如需要在类型运算时将某个还未确定的类型提前使用，可以使用`infer T`，这时`T`就是一个类型变量。举个例子：
    ```ts 
    type ReturnType<(args: any) => any> = {
      T extends (args: any) => infer R? R : any // 获取返回类型
    }
    ```
    ```ts
    type ArgType<(args: any) => any> = {
      T extends (args: infer A) => any? A : any // 获取形参类型
    }
    ```

3. 泛型工具
  Record作用是让K中所有属性都转化成T类型
  ```ts
  type Record<K extends keyof any, T> = {
    [P in K]: T;
  }
  ```
  Partial作用是给把每一个属性都变成可选的
  ```ts
  type Partial<T> = {
    [P in keyof T]?: T[P];
  };
  ```
  Required作用是给把每一个属性都变成必选的
  ```ts
  type Required<T> = {
    [P in keyof T]-?: T[P];
  };
  ```
  Readonly作用是把某对象的属性变成readonly
  ```ts
  type Readonly<T> = {
    +readonly [P in keyof T]: T[P]
  };
  ```
  Pick作用是从某个接口中挑出需要的属性
  ```ts
  type Pick<T, K extends keyof T> = {
    [P in K]: T[P];
  };
  ```
  ReturnType作用是得到函数的返回值
  ```ts
  type ReturnType<(...args: any) => any> = {
    T extends (...args: any) => infer R? R : any
  }
  ```
  Omit作用是从对象中删除一个属性
  ```ts
  type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;
  ```

参考：
[一文读懂 TypeScript 泛型及应用（ 7.8K字）](https://juejin.cn/post/6844904184894980104)
[ts中文网](https://www.tslang.cn/docs/handbook/generics.html)
[你不知道的 TypeScript 泛型（万字长文，建议收藏）](https://lucifer.ren/blog/2020/06/16/ts-generics/)