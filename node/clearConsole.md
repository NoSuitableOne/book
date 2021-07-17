执行下面这行代码可以清空控制台
```js
process.stdout.write(process.platform === 'win32' ? '\x1B[2J\x1B[0f' : '\x1B[2J\x1B[3J\x1B[H');
```
这行代码的出处是`react-dev-utils/clearConsole.js`，如果使用`create-react-app`这个包的时候注意一下就会发现，执行`npm run xxx`命令的时候会先清空控制台，然后再输出日志，实现清空控制台功能就是这行代码。

解释一下这行代码：前半部分就是写一个流，然后判断一下操作系统，关键魔法在后半段
```js
'\x1B[2J\x1B[0f' 
'\x1B[2J\x1B[3J\x1B[H'
```
首先这两行代码叫ANSI Escape Sequence [ANSI转义序列](https://zh.wikipedia.org/wiki/ANSI%E8%BD%AC%E4%B9%89%E5%BA%8F%E5%88%97) 具体定义可以查看维基百科。所有序列都以ASCII字符ESC（27 / 十六进制 0x1B）开头，里面的符号都可以拆分，比如 `[2J`标识清屏，`[H`表示把光标移动到左上角最顶部，把这些连结起来就完成了清屏动作。
