# 上传的解决方案
上传也算前端领域最常见的需求了，总结一下上传组件的解决方案。

## 传统方式
传统上传直接通过html的表单标签就能搞定
```html
<form action="xxx">
  <input type="file" />
</form>
```
问题：ui样式完全靠浏览器自带，也无法新增自己的逻辑

## 接口上传
不绕弯子，接口上传核心代码是headers中`Content-Type': 'multipart/form-data`，替换掉表单提交默认的`Content-Type:application/x-www-form-urlencoded`
这种格式的关键点是报文被分割成多个字段，带文件的file部分会标识
```js
const res = await request({
  api: url,
  method: 'POST',
  headers: {
    'Content-Type': 'multipart/form-data'
  },
  data: data,
  onUploadProgress: this.onProgress
});
```
相应的，后端controller层也需要做一些处理。比如使用Express或者NestJS框架，如果解析参数的方式不匹配框架提供的`req.body`方法会取不到数据。解决方案就是引入body-parser中间件。当然一般情况下，处理上传会直接使用合适的中间件，比如`multer`。

## 上传进度
这是一个很简单的知识点。js的XMLHttpRequest对象原生提供了一种onprogress方法
```js
//进度事件
xhr.onprogress = function(e){
  e = e || event;
  if (e.lengthComputable){
      result.innerHTML = "Received " + e.loaded + " of " + e.total + " bytes";
  }
};
```

## 上传组件的封装
有了上面的基础，基本明白上传的逻辑流程，剩下的是UI组件的封装，参考[我是如何设计 Upload 上传组件的](https://segmentfault.com/a/1190000018205412)的思路，UI组件完全可以设计成包裹各种实现方式的上传组件。
```js
<UploadWrapper 
  multiple={false} 
  accept={['png', 'jpg', 'gif']} 
  api='//localhost:3333/upload/file'
>
  {children}
</UploadWrapper>
```
核心的Uploader封装在UploaderWrapper上，children中只需触发父组件传过来的上传方法即可唤起父组件内的上传。
常用的上传组件分三类：点击、拖拽、粘贴
### 点击
点击上传没什么好说的，包裹一个原生标签即可
### 拖拽
拖拽的核心是拖拽事件的`event.dataTransfer`属性，文件存储在这个属性中传输
### 粘贴
粘贴的核心是粘贴事件的`event.clipboardData`属性，文件存储在这个属性中传输
### 预览
预览的核心在于拿到文件流然后展示，<input type="file">中的fileList， 
`event.clipboardData`、`event.dataTransfer`的值使用new FileReader()都可以读取，调用对应的api可以得到文本内容或url即可展示。

## 断点续传
后面补demo，可以先参考这个解决方案[字节跳动面试官：请你实现一个大文件上传和断点续传](https://juejin.cn/post/6844904046436843527)

参考：
  [rfc2388](https://tools.ietf.org/html/rfc2388)
  [我是如何设计 Upload 上传组件的](https://segmentfault.com/a/1190000018205412)
  [深入理解ajax系列第五篇——进度事件](cnblogs.com/xiaohuochai/p/6552674.html)
  [字节跳动面试官：请你实现一个大文件上传和断点续传](https://juejin.cn/post/6844904046436843527)