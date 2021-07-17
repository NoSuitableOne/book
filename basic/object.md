## descriptor
Object的属性有一个属性描述符
### 查
Object.getOwnProperyDescriptor方法可以获取Object的descriptor
```js
{  
  "value" // 值
  "writable", // 是否只读，控制的是相应键值的value，注意和configurable的对比区分
  "enumerable", // 是否可被遍历
  "configurable" // 键值是否可以被删除、编辑，注意这个值只决定了对象中必须有这个键值，对应的value不被控制
}
 ``` 
### 增
```js
Object.defineProperty(user, "name", {
  value: "John"
});
``` 
