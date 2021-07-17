## 声明文件
首先，如果本来就是ts编写的代码，只需要在编译选项中开启生成声明文件选项`compilerOptions { declaration: true }`即可生成声明文件。
这里讨论为js库编写声明文件的场景：某些js库已经完成逻辑编写，可以直接在js项目中使用，但是没有提供声明文件，所以需要为这些js库编写声明文件才能在ts项目中使用。

为js库编写声明文件需要理解这个核心问题：1. 库生效的方式 2. 三斜线指令 3. 声明文件的写法 4. shime.d.ts
假设有一个处理字符串的js库str-util，如果在ts中直接使用是会报错的，因为引用的这个js库缺少声明文件

### 库生效的方式
#### 引入方式
现阶段，随着各种前端框架的普及，大家用js库多半也是在项目里安装npm包然后引用。但是，如果某些库有现成的cdn资源、可下载到本地的脚本，选择使用`<script>`标签引入也是很常见的方案。所以，针对不同方式，声明文件有不同的写法。关键点在于库的生效方式。
一般生效方式分两大类：
  1. 直接在全局上添加了一个变量，比如JQuery、lodash，一旦引入，window/global下就有了一个新变量。
    对应的方案是注入全局变量，直接新建一个`*.d.ts`文件，然后使用关键词`declare`。
    ```ts
    declare namespace LibName {
      function xxx
      const yyy
      ...
    }
    declare class ClassName {
      ...
    }
    ```
    这样声明都是全局生效的，可以直接使用
    ```ts
    LibName.xxx();
    const instance = new ClassName()
    ```
  2. 库本身就有模块化方案，需要引入模块后使用。这种情形复杂一些，因为js历经了几代标准，模块化方案也一直在变化，不同的模块化方案解决方式不一样。
    - esm
      如果库本身已经是esm就非常简单，仅仅需要把export的方法定义一遍type
    - commonjs
      commonjs规范可以粗浅地理解为只是语法上和esm不同，声明文件写法上区别很小，同样`declare moudule xxx {...}`即可定义类型
    - UMD
      UMD是同时适配script标签引入和npm包模块导入的方案，所以在npm包导入部分可以直接参考前面的情形，稍有区别的只是ts提供了`export as namespace xxx`这样的语法，专门用于全局导出。
### 三斜线指令
三斜线指令是ts早期就给出的一个引入第三方库的方案，现在大部分场景都可以被模块导入替代。
常见的情况是js库依赖另一个全局库，所以ts给了个三斜线指令来完整引入第三方文件。
```ts
///<reference path=“path/to/module” />
///<reference types=“node” /> // 对应nodeModules/@types
```

### 声明文件的写法
这部分直接上几个例子展示如何编写声明文件，来源于[ts exercises](https://typescript-exercises.github.io/#exercise=1&file=%2Findex.ts)
第一例：
```js
/**
 * Reverses a string.
 * @param {String} value
 * @return {String}
 */
function strReverse(value) {
    return value.split('').reverse().join('');
}

/**
 * Converts a string to lower case.
 * @param {String} value
 * @return {String}
 */
function strToLower(value) {
    return value.toLowerCase();
}

/**
 * Converts a string to upper case.
 * @param {String} value
 * @return {String}
 */
function strToUpper(value) {
    return value.toUpperCase();
}

/**
 * Randomizes characters of a string.
 * @param {String} value
 * @return {String}
 */
function strRandomize(value) {
    var array = value.split('');
    for (var i = array.length - 1; i > 0; i--) {
        var j = Math.floor(Math.random() * (i + 1));
        var temp = array[i];
        array[i] = array[j];
        array[j] = temp;
    }
    return array.join('');
}

/**
 * Inverts case of a string.
 * @param {String} value
 * @return {String}
 */
function strInvertCase(value) {
    return value
        .split('')
        .map(function(c) {
            if (c === c.toLowerCase()) {
                return c.toUpperCase();
            } else {
                return c.toLowerCase();
            }
        })
        .join('');
}

module.exports = {
    strReverse,
    strToLower,
    strToUpper,
    strRandomize,
    strInvertCase
};
```
```ts
declare module 'stats' {
    type TComparator<T> = { (a: T, b: T): number };
    type TGetValue<T> = { (a: T): number };
    export function getMaxIndex<T>(input: Array<T>, comparator: TComparator<T>): number;
    export function getMaxElement<T>(input: Array<T>, comparator: TComparator<T>): T | null;
    export function getMinIndex<T>(input: Array<T>, comparator: TComparator<T>): number;
    export function getMinElement<T>(input: Array<T>, comparator: TComparator<T>): T | null;
    export function getMedianIndex<T>(input: Array<T>, comparator: TComparator<T>): number;
    export function getMedianElement<T>(input: Array<T>, comparator: TComparator<T>): T | null;
    export function getAverageValue<T>(input: Array<T>, getValue: TGetValue<T>): number | null
}
```
第二例：
```js
function getMaxIndex(input, comparator) {
    if (input.length === 0) {
        return -1;
    }
    var maxIndex = 0;
    for (var i = 1; i < input.length; i++) {
        if (comparator(input[i], input[maxIndex]) > 0) {
            maxIndex = i;
        }
    }
    return maxIndex;
}

function getMaxElement(input, comparator) {
    var index = getMaxIndex(input, comparator);
    return index === -1 ? null : input[index];
}

function getMinIndex(input, comparator) {
    if (input.length === 0) {
        return -1;
    }
    var maxIndex = 0;
    for (var i = 1; i < input.length; i++) {
        if (comparator(input[maxIndex], input[i]) > 0) {
            maxIndex = i;
        }
    }
    return maxIndex;
}

function getMinElement(input, comparator) {
    var index = getMinIndex(input, comparator);
    return index === -1 ? null : input[index];
}

function getMedianIndex(input, comparator) {
    if (input.length === 0) {
        return -1;
    }
    var data = input.slice().sort(comparator);
    return input.indexOf(data[Math.floor(data.length / 2)]);
}

function getMedianElement(input, comparator) {
    var index = getMedianIndex(input, comparator);
    return index === -1 ? null : input[index];
}

function getAverageValue(input, getValue) {
    if (input.length === 0) {
        return null;
    }
    return input.reduce(
        function (result, item) {
            return result + getValue(item);
        },
        0
    ) / input.length;
}

module.exports = {
    getMaxIndex,
    getMaxElement,
    getMinIndex,
    getMinElement,
    getMedianIndex,
    getMedianElement,
    getAverageValue
};
```
```ts
declare module 'stats' {
    type TComparator<T> = { (a: T, b: T): number };
    type TGetValue<T> = { (a: T): number };
    export function getMaxIndex<T>(input: Array<T>, comparator: TComparator<T>): number;
    export function getMaxElement<T>(input: Array<T>, comparator: TComparator<T>): T | null;
    export function getMinIndex<T>(input: Array<T>, comparator: TComparator<T>): number;
    export function getMinElement<T>(input: Array<T>, comparator: TComparator<T>): T | null;
    export function getMedianIndex<T>(input: Array<T>, comparator: TComparator<T>): number;
    export function getMedianElement<T>(input: Array<T>, comparator: TComparator<T>): T | null;
    export function getAverageValue<T>(input: Array<T>, getValue: TGetValue<T>): number | null
}
```
第三例：
```js
function dateWizard(date, format) {
    var details = dateWizard.dateDetails(date);
    return format.replace(/{([^}]+)}/g, function(s, match) {
        return dateWizard.pad(details[match]);
    });
}

dateWizard.pad = function(s) {
    s = String(s);
    while (s.length < 2) {
        s = '0' + s;
    }
    return s;
};

dateWizard.dateDetails = function dateDetails(date) {
    return {
        year: date.getFullYear(),
        month: date.getMonth() + 1,
        date: date.getDate(),
        hours: date.getHours(),
        minutes: date.getMinutes(),
        seconds: date.getSeconds()
    };
};

dateWizard.utcDateDetails = function getUTCDateDetails(date) {
    return {
        year: date.getUTCFullYear(),
        month: date.getUTCMonth() + 1,
        date: date.getUTCDate(),
        hours: date.getUTCHours(),
        minutes: date.getUTCMinutes(),
        seconds: date.getUTCSeconds()
    };
};

module.exports = dateWizard;
```
```ts
declare function dateWizard(date: Date, format: string): string;

declare namespace dateWizard {
    interface DateDetails {
        year: number;
        month: number;
        date: number;
        hours: number;
        minutes: number;
        seconds: number;
    }

    const pad: (s: number) => string;
    function dateDetails(date: Date): DateDetails;
    function utcDateDetails(date: Date): DateDetails;
}

export = dateWizard;
```
# shims.d.ts
因为ts编译器默认只会处理`.js`和`.ts`文件，所以前端工程中的样式文件（比如`.css`）、资源文件（比如`.png`）、某些框架中特有的文件（比如`.vue`）是无法是别的，引入(比如`import styles from 'xxx.css'`)就会看到报错。shims.d.ts就是解决这个问题。
shims.d.ts本质就是个声明文件，如果使用import引入的文件类型ts不识别，就加一个声明文件让ts当作一个普通模块来处理，例如样式文件可以这样解决
```ts
declare module '*.css' {
  const classes: { readonly [key: string]: string }
  export default classes
}
```

参考：
[Typescript 书写声明文件（可能是最全的）](https://juejin.cn/post/6844904034621456398#heading-5)
[Typescript中文网-三斜线指令](https://www.tslang.cn/docs/handbook/triple-slash-directives.html)
[ts exercises](https://typescript-exercises.github.io/#exercise=1&file=%2Findex.ts)
