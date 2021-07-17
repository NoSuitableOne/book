# 插槽（slot）
插槽分为匿名插槽、具名插槽和作用域插槽
## 匿名插槽
```js
<template>
  <child>
    <div v-slot:default>
      parent value here
    </div>
  </child>
</template>
// child
<template>
  child
  <slot>placeholder</slot>
</template>
```
v-slot的缩写是default

## 具名插槽
具名插槽和匿名插槽是一样的，只不过多个名字
```js
<template>
  <child>
    <div v-slot:slotname>
      parent value here
    </div>
  </child>
</template>
// child
<template>
  child
  <slot name="slotname">placeholder</slot>
</template>
```

## 作用域插槽
作用域插槽的关键是父组件使用子组件不仅要给子组件传值，还要用子组件的数据
```js
<template>
  <child>
    <div v-slot:slotname="childData">
      {{childData.name}}
    </div>
  </child>
</template>
// child
<template>
  child
  <slot name="slotname">
    child default value: {{data.name}}
  </slot>
</template>
<script>
  data() {
    return {
      name: 'name'
    }
  }
</script>
```