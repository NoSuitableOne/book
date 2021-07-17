# vue表单输入
## 基本介绍
vue的表单输入得益于MVVM模式，而非单项数据流比react好用很多（如果用过Angular，不用解释就能明白vue的v-model）。一般表单输入都希望把用户的输入结果直接绑定到数据层，直接完成数据驱动。vue中可以使用v-model,v-model的作用就是直接将选择动作和vm.data绑定。
```js
<select v-model="selected">
  <option :value="{ number: 123 }">123</option>
</select>
<script>
  data() {
    return {
      selected: ''
    }
  }
</script>
```

## 语法糖
v-model的本质是语法糖，这句话如果用过react，估计看到这儿就直接明白后面的内容。
简单解释一下
```js
<input v-model="inputData">
<input :value="inputData" @input="changeData($event.target.value)">
```
上面两个input是等价的，相当于如果不用v-model，就需要自己去绑定value和处理事件

# 修饰符
.lazy @input -> @change
```
<input v-model.lazy="inputData">
```

.number 限定成number类型

.trim trim
