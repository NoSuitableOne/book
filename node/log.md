## log4js
### 最简单的例子
下面这段代码是使用log4js最简单的例子，运行之后将不输出任何内容，这么设定的原因是方便其它库开发者0成本使用（这里有部分文章讲解log4js使用的是早期版本，默认输出到console，这在新版本是不适用的）。
```js
var log4js = require('log4js');
var logger = log4js.getLogger();
logger.debug('Time: ', new Date());
```
想要输出内容，设置logger.level即可，设置level表示高于这个level的日志会被输出
```js
logger.level = 'DEBUG';
```

### log4js.configure()
log4js.configure接受2种参数类型：
  1. string表示配置文件路径
  2. 直接传入json配置对象

configure对象接口如下
```ts
export interface Configuration {
  appenders: { [name: string]: Appender; };
  categories: { [name: string]: { appenders: string[]; level: string; enableCallStack?: boolean; } };
  pm2?: boolean;
  pm2InstanceVar?: string;
  levels?: Levels;
  disableClustering?: boolean;
}
```  
 - appenders: 可接受的`appender`类型很多，最常用的是`console`、`stdout`、`file`、`dataFile`，所有可支持的配置包括(
    categoryFilter
    console
    dateFile
    file
    fileSync
    logLevelFilter
    multiFile
    multiprocess
    recording
    stderr
    stdout
    tcp
    tcp-server
  )。日志名称可以接受一个`pattern`参数，这个参数支持时间格式，例如`pattern：.yyyy-MM-dd`会生成一个带有时间戳的日志文件，设置`compress：true`则会将前面时段的日志文件压缩
 - categories: 表示日志的类型分类，注册的类型供`log4js.getLogger`方法调用，默认类型是`default`。比如上面的简单例子等同于
    ```js
    var log4js = require('log4js');
    log4js.configure({
      appenders: {
        app: { type: "console" }
      },
      categories: {
        default: { appenders: ['app'], level: 'DEBUG' }
      }
    });
    var logger = log4js.getLogger(); // 这里使用log4js.getLogger(‘default’)也是一样的
    logger.debug('Time: ', new Date());
    ```
早期版本中有一个`replaceConsole`的属性，现阶段这个属性已经被废弃，官方提供的trick是
```js
log4js.configure(...); // set up your categories and appenders
const logger = log4js.getLogger('console');
console.log = logger.info.bind(logger); // do the same for others - console.debug, etc.
```

## koa中使用
### koa-log4js
koa中有个适配log4js的中间件叫koa-log4js，这是一个基于log4js的koa中间件，所以使用上和log4js没有差别。
```js
const log4js = require('koa-log4');
app.use(log4js.koaLogger(log4js.getLogger("http"), { level: 'auto' }))
```
### 自己封装中间件
如果需要自己封装一个类似的中间件也很简单
以`hello-world`例子为基础
```js
const Koa = require('koa');
const app = new Koa();

app.use(async function(ctx) {
  ctx.body = 'Hello World';
});
app.listen(3000);

module.exports = app;
```

依照官方示例，增加一个logger中间件即可每次请求时打印日志
```js
const Koa = require('koa');
...

app.use(async (ctx, next) => {
  const start = Date.now();
  await next();
  const ms = Date.now() - start;
  console.log(`${ctx.method} ${ctx.url} - ${ms}ms`);
});

app.use(async function(ctx) {
  ...
});

...
```

如果把日志替换成log4js即完成了中间件的封装
```js
...
const log4js = require('log4js');

app.use(async (ctx, next) => {
  const start = Date.now();
  await next();
  const ms = Date.now() - start;
  const logger = log4js.getLogger();
  logger.level = 'DEBUG';
  logger.info(`${ctx.method} ${ctx.url} - ${ms}ms`);
});
...
```

当然，实际使用中，一般会把log4js的配置接口暴露出来，让用户按自己需求设置
先自行定义一个logger中间件
```js
function koaLogger (configure, categories) {
  return async (ctx, next) => {
    const start = Date.now();
    log4js.configure(configure);
    const loggerCategory = {};
    categories.map(category => loggerCategory[category] = log4js.getLogger(category));
    ctx.loggerCategory = loggerCategory;
    await next();
    const ms = Date.now() - start;
    ctx.loggerCategory['default'].info(`${ctx.method} ${ctx.url} - ${ms}ms`);
  };
};
```
然后初始化中间件，并使用
```js
// log4js的配置
const loggerConfig = {
  appenders: {
    app: { type: "console" },
    access: { type: "file", filename: 'access.log' }
  },
  categories: {
    default: { appenders: ['app'], level: 'DEBUG' },
    monitor: { appenders: ['app', 'access'], level: 'INFO' }
  }
}
// 初始化koa-logger中间件
loggerMiddleware = koaLogger(loggerConfig, ['default', 'monitor']);
// 注册中间件
app.use(loggerMiddleware);
```
通过ctx传递，也可以在洋葱模型的后续中间件中使用logger
```js
app.use(async function(ctx) {
  ctx.loggerCategory['monitor'].info(`reqUrl: ${ctx.req.url} - httpMethod: ${ctx.req.method}`);
  ctx.body = 'Hello World';
});
```
至此，就完成了一个最基本的koa-log4js-middleware封装

## NestJS中打造日志系统
### NestJS的自带logger
NestJS有一个全局开关，可以直接禁用所有自带logger，或者设定logger的输出level，相对高级的用法还可以直接设置logger对象为自定义的LoggerService
```js
const app = await NestFactory.create(AppModule, {
  logger: false //  ['error', 'warn']
});
```

```js
import { LoggerService } from '@nestjs/common';
class myLogger implements LoggerService {
  ...
}

const app = await NestFactory.create(AppModule, {
  logger: new myLogger()
});
```
### NestJS接入log4js
NestJS本身在`@nest/common`下提供了日志类和日志接口，如果要使用非自带的日志，就是要实现接口或者继承原生的日志类
下面是继承Logger类的方法，实现`LoggerService`接口也是可以的，因为NestJS自带的`logger`类也是实现了`LoggerService`接口
```js
import { Logger, Injectable } from '@nestjs/common';
import {
  Logger as Log4jsLogger,
  configure,
  Configuration,
  getLogger,
} from 'log4js';

@Injectable()
export class AppLogger extends Logger {
  log4js: Log4jsLogger;

  constructor() {
    super();
    this.log4js = this.create({
      appenders: {
        app: {
          type: 'console',
        },
        access: {
          type: 'file',
          filename: 'access.log',
        },
      },
      categories: {
        default: {
          appenders: ['app'],
          level: 'DEBUG',
        },
        monitor: {
          appenders: ['app', 'access'],
          level: 'INFO',
        },
      },
    });
  }

  private create(config: string | Configuration): Log4jsLogger {
    if (typeof config === 'string') {
      configure(config);
    } else {
      configure(config);
    }
    return getLogger('monitor');
  }

  log(message: string) {
    super.log(message);
    this.log4js.log(message);
  }
  error(message: string, trace: string) {
    super.log(message, trace);
    this.log4js.error(message, trace);
  }
  warn(message: string) {
    super.warn(message);
    this.log4js.warn(message);
  }
  debug(message: string) {
    super.debug(message);
    this.log4js.debug(message);
  }
  verbose(message: string) {
    super.verbose(message);
    this.log4js.info(message);
  }
}
```
实现完日志接口需要完成依赖注入，因为NestJS的create执行是在所有module之外的，所以NestJS在全局提供了一种在`NestFactory.create`后，直接在app实例上挂载日志的实例方法
```js
app.useLogger(app.get(AppLogger));
```
至此，就完成了用log4js替代NestJS自带的log。

### 更全面的日志
除了上面的日志以外，根据NestJS的运行周期，还可以在`Exception Filter`中使用日志，记录服务器内部报错，在`Controller`层的`Interceptors`中拦截resquest和response，并记入日志。


#### Exception Filter
按照官方文档，可以实现`ExceptionFilter`接口扩展exception层的功能，所以直接在这层加入日志，即可记录服务器内部错误
```js
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { AppLogger } from './appLogger.service';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();
    const logger = new AppLogger();
    logger.error(`request: ${request.url}`, `exception: ${exception}`);

    const status =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}
```
当然，和前面一样，这里也需要在app实例方法上挂载新的`exceptionFilter`
```js
app.useGlobalFilters(new AllExceptionsFilter());
```

#### Interceptors
拦截器的概念在纯前端的框架中也有，比如有名的`Axios`，所以相对比较好理解。NestJS也提供了类似的拦截器，所以在每个拦截器里加上日志是可以方便记录response和request的。
一切类似，自己实现拦截器接口，然后在controller上依赖注入，或者是挂载到app的实例方法上即可。
```js
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';
import { AppLogger } from './appLogger.service';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const logger = new AppLogger();
    logger.log('Before...');

    const now = Date.now();
    return next
      .handle()
      .pipe(tap(() => logger.log(`After... ${Date.now() - now}ms`)));
  }
}
```
controller
```js
@UseInterceptors(LoggingInterceptor)
export class CatsController {}
```
