---
# 主题列表：juejin, github, smartblue, cyanosis, channing-cyan, fancy, hydrogen, condensed-night-purple, greenwillow, v-green, vue-pro, healer-readable, mk-cute, jzman, geek-black, awesome-green, qklhk-chocolate
# 贡献主题：https://github.com/xitu/juejin-markdown-themes
theme: github
highlight: github
---

## 浅谈BFF

最近我们后端伙伴开始采用了微服务架构，拆分了很多领域服务，身为大前端的我们肯定也要做出改变，平常一个列表需要一个接口就能拿到数据，但微服务架构下就需要中间有一层专门为前端聚合微服务架构下的n个接口，方便前端调用，于是我们就采用了当下比较流行的BFF方式。

bff和node没有强绑定关系，但让前端人员去熟悉node之外的后端语言学习成本太高，所以技术栈上我们使用node作为中间层，node的http框架我们使用的是nestjs。

### BFF作用

BFF（Backends For Frontends），就是服务于前端的后端，经过几个项目的洗礼，我对它也有了一些见解，我认为它主要有以下作用：

- 接口聚合和透传：和上文所讲的一致，聚合多个接口，方便前端调用
- 接口数据格式化：前端页面只负责 UI 渲染和交互，不处理复杂的数据关系，前端的代码可读性和可维护性会得到改善
- 减少人员协调成本：后端微服务和大前端bff落地并且完善后，后期部分需求只需要前端人员开发即可


### 适用场景

BFF虽然比较流行，但不能为了流行而使用，要满足一定的场景并且**基建**很完善的情况下才使用，否则只会增加项目维护成本和风险，收益却非常小，我认为的适用场景如下：

- 后端有稳定的领域服务，需要聚合层
- 需求变化频繁，接口经常需要变动：后端有一套稳定的领域服务为多个项目服务，变动的话成本较高，而bff层针对单一的项目，在bff层变动可以实现最小成本的改动。
- 有完善的基建：日志，链路，服务器监控，性能监控等（必备条件）


## Nestjs

本文我就以一名纯前端入门后端的小白的视角来介绍一下Nestjs。
> Nest 是一个用于构建高效，可扩展的 Node.js 服务器端应用程序的框架

### 前端发起请求后后端是怎么做的

首先我们发起一个GET请求


```js
fetch('/api/user')
    .then(res => res.json())
    .then((res) => {
    	// do some thing
    })
```

假设nginx的代理已经配置好（所有`/api`开头的请求都到我们的bff服务），后端会接收到我们的请求，那么问题来了，它是通过什么接收的？

首先我们初始化一个Nestjs的项目，并创建user目录，它的目录结构如下

```sh
├── app.controller.ts # 控制器
├── app.module.ts # 根模块
├── app.service.ts # 服务
├── main.ts # 项目入口，可以选择平台、配置中间件等
└── src 业务模块目录
	├── user
    		├── user.controller.ts
    		├── user.service.ts
    		├── user.module.ts
```

Nestjs是在`Controller`层通过路由接收请求的，它的代码如下：

`user.controller.ts`

```ts
import {Controller, Get, Req} from '@nestjs/common';

@Controller('user')
export class UserController {
  @Get()
  findAll(@Req() request) {
    return [];
  }
}
```

在这里先说明一下Nestjs的一些基础知识
使用Nestjs完成一个基本服务需要有`Module`,`Controller`,`Provider`三大部分。

- `Module`，字面意思是模块，在nestjs中由`@Module()`修饰的class就是一个Module，在具体项目中我们会将其作为**当前子模块的入口**，比如一个完整的项目可能会有用户模块，商品管理模块，人员管理模块等等。
- `Controller`，字面意思是控制器，负责处理客户端传入的请求和服务端返回的响应，官方定义是一个由`@Controller()`修饰的类，上述代码就是一个Controller，当我们发起地址为`'/api/user'`的get请求的时候，Controller就会定位到`findAll`的方法，这个方法的返回值就是前端接收到的数据。
- `Provider`，字面意思是提供者，其实就是为Controller提供服务的，官方的定义是由`@Injectable()`修饰的class，我简单解释一下：上述代码直接在Controller层做业务逻辑处理，后续随着业务迭代，需求越来越复杂，这样的代码会难以维护，**所以需要一层来处理业务逻辑**，Provider正是这一层，它需要`@Injectable()`修饰。

我们再来完善一下上面的代码，增加`Provider`,在当前模块下创建`user.service.ts`

`user.service.ts`

```ts
import {Injectable} from '@nestjs/common';

@Injectable()
export class UserService {
    async findAll(req) {
        return [];
    }
}
```
然后我们的Controller需要做一下更改

`user.controller.ts`

```ts
import {Controller, Get, Req} from '@nestjs/common';
import {UserService} from './user.service';

@Controller('user')
export class UserController {
    constructor(
        private readonly userService: UserService
    ) {}

  @Get()
    findAll(@Req() request) {
        return this.userService.findAll(request);
    }
}
```
这样我们的Controller和Provider就完成了，两层各司其职，代码可维护性增强。

接下来，我们还需要将Controller和Provider注入到Module中，我们新建一个`user.module.ts`文件，编写以下内容:

`user.module.ts`

```ts
import {Module} from '@nestjs/common';
import {UserController} from './user.controller';
import {UserService} from './user.service';

@Module({
    controllers: [UserController],
    providers: [UserService]
})
export class UsersModule {}
```

这样，我们的一个业务模块就完成了，剩下只需要将`user.module.ts`引入到项目总模块注入一下，启动项目后，访问'/api/user'就能获取到数据了，代码如下：

`app.module.ts`
```ts
import {Module} from '@nestjs/common';
import {APP_FILTER} from '@nestjs/core';
import {AppController} from './app.controller';
import {AppService} from './app.service';
import {UsersModule} from './users/users.module';

@Module({
    // 引入业务模块
    imports: [UsersModule],
    controllers: [AppController],
    providers: [
        AppService
    ]
})
export class AppModule {}
```

### Nestjs常用模块

通过阅读上文我们了解了跑通一个服务的流程和nestjs的接口是如何相应数据的，但还有很多细节没有讲，比如大量装饰器(`@Get`,`@Req`等)的使用，下文将为大家讲解Nestjs常用的模块

- 基础功能
	- Controller 控制器
    - Provider 提供者（业务逻辑）
    - Module 一个完整的业务模块
    - NestFactory 创建 Nest 应用的工厂类
- 高级功能
	- Middleware 中间件
    - Exception Filter 异常过滤器
    - Pipe 管道
    - Guard 守卫
    - Interceptor 拦截器
    
Controller、Provider、Module上文中已经提过，这里就不进行二次讲解，NestFactory其实就是用来创建一个Nestjs应用的一个工厂函数，通常在入口文件来创建，也就是上文目录中的main.ts，代码如下：

`main.ts`

```ts
import {NestFactory} from '@nestjs/core';
import {AppModule} from './app.module';

async function bootstrap() {
    const app = await NestFactory.create(AppModule);
    await app.listen(3000);
}
bootstrap();

```

#### decorator 装饰器

装饰器是Nestjs中常用的功能，它内部提供了一些常用的请求体的装饰器，我们也可以自定义装饰器，你可以在任何你想要的地方很方便地使用它。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2cd0d5a67c9146bfb6bd9e126a4c5e06~tplv-k3u1fbpfcp-watermark.image)

除了上面这些之外，还有一些修饰class内部方法的装饰器，最常见的就是`@Get()`,`@Post()`,`@Put()`,`@Delete()`等路由装饰器，我相信绝大多数前端都可以看明白这些什么意思，就不再解释了。

#### Middleware 中间件
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/37b046898c134b7cb08191dec5f7d8e9~tplv-k3u1fbpfcp-watermark.image)

Nestjs是对Express的二次封装，Nestjs中的中间件等价于[Express中的中间件](https://expressjs.com/zh-cn/guide/using-middleware.html)，最常用的场景就是**全局的日志、跨域、错误处理、cookie格式化**等较为常见的api服务应用场景，官方解释如下：

> 中间件函数能够访问请求对象 (req)、响应对象 (res) 以及应用程序的请求/响应循环中的下一个中间件函数。下一个中间件函数通常由名为 next 的变量来表示。

我们以cookie格式化为例，修改后的main.ts的代码如下：

```ts
import {NestFactory} from '@nestjs/core';
import * as cookieParser from 'cookie-parser';
import {AppModule} from './app.module';

async function bootstrap() {
    const app = await NestFactory.create(AppModule);
    // cookie格式化中间件，经过这个中间件处理，我们就能在req中拿到cookie对象
    app.use(cookieParser());
    await app.listen(3000);
}
bootstrap();

```

#### Exception Filter 异常过滤器

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b05a8b347df3426fb2561afbbff0fa53~tplv-k3u1fbpfcp-watermark.image)
Nestjs内置异常层，内置的异常层负责处理整个应用程序中的所有抛出的异常。当捕获到未处理的异常时，最终用户将收到友好的响应。

身为前端的我们肯定收到过接口报错，异常过滤器就是负责抛出报错的，通常我们项目需要自定义报错的格式，和前端达成一致后形成一定的接口规范。内置的异常过滤器给我们提供的格式为：

```json
{
  "statusCode": 500,
  "message": "Internal server error"
}
```
一般情况这样的格式是不满足我们的需求的，所以我们需要**自定义异常过滤器并绑定到全局**，下面我们先实现一个简单的异常过滤器：

我们在此项目的基础上增加一个common文件夹，里面存放一些过滤器，守卫，管道等，更新后的目录结构如下：
```sh
├── app.controller.ts # 控制器
├── app.module.ts # 根模块
├── app.service.ts # 服务
├── common 通用部分
├	├── filters
├	├── pipes
├	├── guards
├	├── interceptors
├── main.ts # 项目入口，可以选择平台、配置中间件等
└── src 业务模块目录
	├── user
    		├── user.controller.ts
    		├── user.service.ts
    		├── user.module.ts
```

我们在filters目录下增加http-exception.filter.ts文件

`http-exception.filter.ts`

```ts
import {ExceptionFilter, Catch, ArgumentsHost, HttpException} from '@nestjs/common';
import {Response} from 'express';

// 需要Catch()修饰且需要继承ExceptionFilter
@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
    // 过滤器需要有catch(exception: T, host: ArgumentsHost)方法
    catch(exception: HttpException, host: ArgumentsHost) {
        const ctx = host.switchToHttp();
        const response = ctx.getResponse<Response>();
        const status = exception.getStatus();
        const msg = exception.message;

        // 这里对res的处理就是全局错误请求返回的格式
        response
            .status(status)
            .json({
                status: status,
                code: 1,
                msg,
                data: null
            });
    }
}
```

接下来我们绑定到全局,我们再次更改我们的app.module.ts

`app.module.ts`

```ts
import {Module} from '@nestjs/common';
import {APP_FILTER} from '@nestjs/core';
import {AppController} from './app.controller';
import {AppService} from './app.service';
import {HttpExceptionFilter} from './common/filters/http-exception.filter';
import {UsersModule} from './users/users.module';

@Module({
    // 引入业务模块
    imports: [UsersModule],
    controllers: [AppController],
    providers: [
        // 全局异常过滤器
        {
            provide: APP_FILTER,
            useClass: HttpExceptionFilter,
        },
        AppService
    ]
})
export class AppModule {}
```

这样我们初始化的项目就有了自定义的异常处理。

#### Pipe 管道
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c82a09e919684b1393589d23b0cc5df4~tplv-k3u1fbpfcp-watermark.image)

这部分单从名称上看很难理解，但是从作用和应用场景上却很好理解，根据我的理解，**管道就是在Controllor处理之前对请求数据的一些处理程序**。


通常管道有两种应用场景：

- **请求数据转换**
- **请求数据验证**：对输入数据进行验证，如果验证成功继续传递; 验证失败则抛出异常

数据转换应用场景不多，这里只讲一下数据验证的例子，数据验证是中后台管理项目最常见的场景。

通常我们的Nest的应用会配合[class-validator](https://github.com/typestack/class-validator)来进行数据验证，我们在pipes目录下新建validation.pipe.ts

`validation.pipe.ts`

```ts
import {PipeTransform, Injectable, ArgumentMetadata, BadRequestException} from '@nestjs/common';
import {validate} from 'class-validator';
import {plainToClass} from 'class-transformer';

// 管道需要@Injectable()修饰，可选择继承Nest内置管道PipeTransform
@Injectable()
export class ValidationPipe implements PipeTransform<any> {
    // 管道必须有transform方法，这个方法有两个参数，value ：当前处理的参数， metadata：元数据
    async transform(value: any, {metatype}: ArgumentMetadata) {
        if (!metatype || !this.toValidate(metatype)) {
            return value;
        }
        const object = plainToClass(metatype, value);
        const errors = await validate(object);
        if (errors.length > 0) {
            throw new BadRequestException('Validation failed');
        }
        return value;
    }

    private toValidate(metatype: Function): boolean {
        const types: Function[] = [String, Boolean, Number, Array, Object];
        return !types.includes(metatype);
    }
}

```

然后我们在全局绑定这个管道，修改后的app.module.ts内容如下：
```ts
import {Module} from '@nestjs/common';
import {APP_FILTER, APP_PIPE} from '@nestjs/core';
import {AppController} from './app.controller';
import {AppService} from './app.service';
import {HttpExceptionFilter} from './common/filters/http-exception.filter';
import {ValidationPipe} from './common/pipes/validation.pipe';
import {UsersModule} from './users/users.module';

@Module({
    // 引入业务模块
    imports: [UsersModule],
    controllers: [AppController],
    providers: [
        // 全局异常过滤器
        {
            provide: APP_FILTER,
            useClass: HttpExceptionFilter,
        },
        // 全局的数据格式验证管道
        {
            provide: APP_PIPE,
            useClass: ValidationPipe,
        },
        AppService
    ]
})
export class AppModule {}
```
这样，我们的应用程序就加入了数据校验功能，比如我们编写需要数据验证的接口，我们需要先新建一个createUser.dto.ts的文件，内容如下：

```ts
import {IsString, IsInt} from 'class-validator';

export class CreateUserDto {
  @IsString()
  name: string;

  @IsInt()
  age: number;
}
```

然后我们在**Controller**层引入，代码如下：

`user.controller.ts`

```ts
import {Controller, Get, Post, Req, Body} from '@nestjs/common';
import {UserService} from './user.service';
import * as DTO from './createUser.dto';


@Controller('user')
export class UserController {
    constructor(
        private readonly userService: UserService
    ) {}

    @Get()
    findAll(@Req() request) {
        return this.userService.findAll(request);
    }

    // 在这里添加数据校验
    @Post()
    addUser(@Body() body: DTO.CreateUserDto) {
        return this.userService.add(body);
    }
}

```

**如果客户端传递过来参数不符合规范，该请求讲直接抛错，不会继续处理。**

#### Guard 守卫

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cd16b448f64f4130bdcad13a92c9b0a6~tplv-k3u1fbpfcp-watermark.image)

守卫，其实就是路由守卫，就是保护我们写的接口的，最常用的场景就是**接口的鉴权**，通常情况下对于一个业务系统每个接口我们都会有登录鉴权，所以通常情况下我们会封装一个全局的路由守卫，我们在项目的common/guards目录下新建auth.guard.ts，代码如下：

`auth.guard.ts`

```ts
import {Injectable, CanActivate, ExecutionContext} from '@nestjs/common';
import {Observable} from 'rxjs';

function validateRequest(req) {
    return true;
}

// 守卫需要@Injectable()修饰而且需要继承CanActivate
@Injectable()
export class AuthGuard implements CanActivate {
    // 守卫必须有canActivate方法，此方法返回值类型为boolean
    canActivate(
        context: ExecutionContext,
    ): boolean | Promise<boolean> | Observable<boolean> {
        const request = context.switchToHttp().getRequest();
        // 用于鉴权的函数，返回true或false
        return validateRequest(request);
    }
}

```

然后我们将它绑定到全局module，修改后的app.module.ts内容如下：

```ts
import {Module} from '@nestjs/common';
import {APP_FILTER, APP_PIPE, APP_GUARD} from '@nestjs/core';
import {AppController} from './app.controller';
import {AppService} from './app.service';
import {HttpExceptionFilter} from './common/filters/http-exception.filter';
import {ValidationPipe} from './common/pipes/validation.pipe';
import {AuthGuard} from './common/guards/auth.guard';
import {UsersModule} from './users/users.module';

@Module({
    // 引入业务模块
    imports: [UsersModule],
    controllers: [AppController],
    providers: [
        // 全局异常过滤器
        {
            provide: APP_FILTER,
            useClass: HttpExceptionFilter,
        },
        // 全局的数据格式验证管道
        {
            provide: APP_PIPE,
            useClass: ValidationPipe,
        },
        // 全局登录鉴权守卫
        {
            provide: APP_GUARD,
            useClass: AuthGuard,
        },
        AppService
    ]
})
export class AppModule {}
```

这样，我们的应用就多了全局守卫的功能

#### Interceptor 拦截器
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5a4964e187844a3ea4fa3a9071174303~tplv-k3u1fbpfcp-watermark.image)

从官方图上可以看出，拦截器可以拦截请求和响应，所以又分为请求拦截器和响应拦截器，前端目前很多流行的请求库也有这一个功能，比如axios，umi-request等，相信前端同学都接触过，其实就是在客户端和路由之间处理数据的程序。

拦截器具有一系列有用的功能，它们可以：

- 在函数执行之前/之后绑定额外的逻辑
- 转换从函数返回的结果
- 转换从函数抛出的异常
- 扩展基本函数行为
- 根据所选条件完全重写函数 (例如, 缓存目的)

下面我们实现一个响应拦截器来格式化全局响应的数据，在/common/interceptors目录下新建res.interceptors.ts文件，内容如下：

`res.interceptors.ts`

```ts
import {Injectable, NestInterceptor, ExecutionContext, CallHandler} from '@nestjs/common';
import {Observable} from 'rxjs';
import {map} from 'rxjs/operators';

export interface Response<T> {
    code: number;
    data: T;
}

@Injectable()
export class ResInterceptor<T> implements NestInterceptor<T, Response<T>> {

    intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
        return next.handle().pipe(map(data => {
            const ctx = context.switchToHttp();
            const response = ctx.getResponse();
            response.status(200);
            const res = this.formatResponse(data) as any;
            return res;
        }));
    }

    formatResponse<T>(data: any): Response<T> {
        return {code: 0, data};
    }
}
```
这个响应守卫的作用就是将我们的接口返回数据格式化成`{code, data}`的格式，接下来我们需要将这个守卫绑定到全局，修改后的app.module.ts内容如下：

```ts
import {Module} from '@nestjs/common';
import {APP_FILTER, APP_PIPE, APP_GUARD, APP_INTERCEPTOR} from '@nestjs/core';
import {AppController} from './app.controller';
import {AppService} from './app.service';
import {HttpExceptionFilter} from './common/filters/http-exception.filter';
import {ValidationPipe} from './common/pipes/validation.pipe';
import {AuthGuard} from './common/guards/auth.guard';
import {ResInterceptor} from './common/interceptors/res.interceptors';
import {UsersModule} from './users/users.module';

@Module({
    // 引入业务模块
    imports: [UsersModule],
    controllers: [AppController],
    providers: [
        // 全局异常过滤器
        {
            provide: APP_FILTER,
            useClass: HttpExceptionFilter,
        },
        // 全局的数据格式验证管道
        {
            provide: APP_PIPE,
            useClass: ValidationPipe,
        },
        // 全局登录鉴权守卫
        {
            provide: APP_GUARD,
            useClass: AuthGuard,
        },
        // 全局响应拦截器
        {
            provide: APP_INTERCEPTOR,
            useClass: ResInterceptor,
        },
        AppService
    ]
})
export class AppModule {}

```

这样，我们这个应用的所有接口的响应格式都固定了。


### Nestjs小总结
经过上文的一系列步骤，我们已经搭建了一个小应用（没有日志和数据源），那么问题来了，前端发起请求后我们实现的应用内部是如何一步步处理并且响应数据的？步骤如下：
> 客户端请求 -> Middleware 中间件 -> Guard 守卫 -> 请求拦截器（我们这没有）-> Pipe 管道 -> Controllor层的路由处理函数 -> 响应拦截器 -> 客户端响应

其中Controllor层的路由处理函数会调用Provider，Provider负责获取底层数据并处理业务逻辑；异常过滤器会在这个程序抛错后执行。

## 总结
经过上文我们可以对BFF层的概念有一个基本的了解，并且按照步骤可以自己搭建一个Nestjs小应用，但和企业级应用差距还很大。
> 企业级应用还需要接入数据源（后端接口数据、数据库数据、apollo配置数据）、日志、链路、缓存、监控等必不可少的功能。

- 接BFF层需要有完善的基建和合适的业务场景，不要盲目接入
- Nestjs基于Express实现，参考了springboot的设计思想，入门很简单，精通需要理解其原理，尤其是**依赖注入**的设计思想

## 参考文献

- [我理解的BFF](https://www.yuque.com/huajinbo/lxhzqg/yrqmtr#vDgjL)
- [NestJs官方文档](https://docs.nestjs.com/)
