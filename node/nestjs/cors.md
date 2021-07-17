NestJs中解决跨域问题

支持全剧跨域的两种方法，其中第二种方法的自由度更高，可以传入一个函数进行适配处理。
```js
const app = await NestFactory.create(AppModule);
app.enableCors();
await app.listen(3000);
```

```js
const app = await NestFactory.create(AppModule, { cors: true });
await app.listen(3000);
```