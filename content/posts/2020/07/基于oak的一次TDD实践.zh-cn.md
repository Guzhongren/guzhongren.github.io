---
title: "基于oak的一次TDD实践"
date: 2020-07-24T22:21:11+08:00
draft: false
author: "谷中仁"
authorLink: "https://guzhongren.github.io"
description: ""
license: "Creative Commons Attribution 4.0 International license"

tags: ["deno", "oak", "tdd", "git", "mvc", "fetch"]
categories: ["deno"]
featuredImage: "https://images.pexels.com/photos/2881785/pexels-photo-2881785.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=750&w=1260"
images: [""]
---

## 简介

Deno 是ry(Ryan Dahl)的新项目，近期发布了其 1.0.0 版，在开发圈子里掀起了不小的风浪，与之创建的 Node 运行时有异曲同工之妙，`真香定律`又一次出现了。

在软件开发中，为了开发出可维护，高质量的程序，使用`TDD`开发可以有效提升项目质量和开发效率。

在这篇博客中，我将使用`Deno`, `Typescript`, `PostgreSql`来开发一个用户管理的 `API` 接口。

## Deno & oak

下面都是来自官网的介绍，写的很通俗易懂，就不用我来解读了。

### Deno

> Deno 是一个简单、现代且安全的 JavaScript 和 TypeScript 运行时环境，其基于 V8 引擎并采用 Rust 编程语言构建。
> * 默认安全设置。除非 显式开启，否则没有文件、网络，也不能访问运行环境。
> * 天生支持 TypeScript。
> * 只有一个单一的可执行文件。
> * 自带实用工具，例如依赖检查器（deno info）和 代码格式化工具（deno fmt）。
> * 有一套经过审核（审计）的标准模块， 确保与 Deno 兼容： deno.land/std

### [oak](https://github.com/oakserver/oak)

> A middleware framework for Deno's net server 🦕

`oak` 是借鉴 Node 框架`Koa`的设计思路开发的一个高性能的框架，其`洋葱模型`式的中间件等思路在开发中使用起来也是非常方便。

## 目标

基于对以上的基础知识的认识，我们计划开发一个用户管理的`API`平台；对于后端简单来说，就是提供关于用户的增删改查（`CURD`）操作。所以我们的主要目标就是提供4个对用户`CURD`的接口。

## 工具

> 工欲善其事，必先利其器。

### 开发工具

[`VS Code`](https://code.visualstudio.com/), [`Docker`](https://www.docker.com/)

### 环境工具

[`Deno`](https://deno.land/), [`Typescript`](https://www.typescriptlang.org/), [`Node`](https://nodejs.org/)

> 注： Node 是用来调试 Deno的

## 基础环境搭建

关于上面开发工具和环境工具的安装就不在这赘述了。

我的环境信息如下：

```shell
❯ node -v
v12.13.0

❯ deno --version
deno 1.2.0
v8 8.5.216
typescript 3.9.2
```

### 项目结构

```shell
❯ tree -L 1 web-api-based-deno
web-api-based-deno
├── .github         // github action
├── .vscode         // debug 及 vscode 配置文件
├── LICENSE         // 仓库许可
├── README.md       // 项目说明，包括数据库连接，简化后的运行命令等
├── _resources      // 基础资源
│   ├── IaaS        // 基础设施，docker-compose 启动postgresql
│   ├── httpClient  // http请求测试
│   └── migration   // 负责生成数据库表
├── deps.ts         // 项目依赖的库及项目中要用到的资源（import）
├── lock.json       // 完整性检查与锁定文件, 参考：https://nugine.github.io/deno-manual-cn/linking_to_external_code/integrity_checking.html
├── makefile        // 将开发需要的命令行简化后目录
├── src             // 源代码目录
└── tests           // 测试目录

5 directories, 5 files

```

## 实现过程

> 先说明一哈，如果要用文字写完整个开发过程个人认为是没有必要的，所以就以最开始的`health`和`addUser`(post接口)为例， 其他接口请参考代码实现。

### 启动基础设施(数据库)并初始化数据表

#### 启动数据库

```shell
❯ make db
cd ./_resources/Iaas && docker-compose up -d
Starting iaas_db_1 ... done
Starting iaas_pgadmin_1 ... done
```

#### 登录`pgadmin`, 在默认的数据库`postgres`中新建Query并执行如下操作，完成初始化数据库。

```sql
CREATE TABLE public."user"
(
    id uuid NOT NULL,
    username character varying(50)  NOT NULL,
    registration_date timestamp without time zone,
    password character varying(20)  NOT NULL,
    deleted boolean
);
```

### src 最终目录

```shell
❯ tree -a -L 4 src
src
├── Utils
│   └── client.ts
├── config.ts
├── controllers
│   ├── UserController.ts
│   ├── health.ts
│   └── model
│       └── IResponse.ts
├── entity
│   └── User.ts
├── exception
│   ├── InvalidedParamsException.ts
│   └── NotFoundException.ts
├── index.ts
├── middlewares
│   ├── error.ts
│   ├── logger.ts
│   └── time.ts
├── repositories
│   └── userRepo.ts
├── router.ts
└── services
    ├── UserService.ts
    └── fetchResource.ts

8 directories, 16 files
```

在开始之前，我们先定义一些常用的结构体和对象，如: response，exception 等

```ts
// src/controllers/model/IResponse.ts
export default interface IResponse {
  success: boolean; // 表示此次请求是否成功
  msg?: String;     // 发生错误时的一些日志信息
  data?: any;       // 请求成功时返回给前端的数据
}
```

```ts
// src/entity/User.ts
export default interface IUser {
  id?: string;
  username?: string;
  password?: string;
  registrationDate?: string;
  deleted?: boolean;
}
export class User implements IUser {}

```

```ts
// src/exception/InvalidedParamsException.ts
export default class InvalidedParamsException extends Error {
  constructor(message: string) {
    super(`Invalided parameters, please check, ${message}`);
  }
}
```

```ts
// src/exception/NotFoundException.ts
export default class NotFoundException extends Error {
  constructor(message: string) {
    super(`Not found resource, ${message}`);
  }
}

```

### 依赖管理

Deno 没有像 Node 一样的诸如package.json来管理依赖，因为Deno的依赖是去中心化的，也就是以远程文件作为库，这一点和`Golang`很像。

我将系统中用到的依赖存放在根目录的`deps.ts`中，在最终提交的时候做一次`[完整性检查与锁定文件](https://nugine.github.io/deno-manual-cn/linking_to_external_code/integrity_checking.html)`, 来保证我所有的依赖在与其他协作者之间是相同的。

首先导入用到的测试相关的依赖。**在后面开发中用到的相关依赖请自行添加到本文件中。** 比较重要的我会列出来。

```ts
export {
  assert,
  equal,
} from "https://deno.land/std/testing/asserts.ts";

```

### 测试先行

现在`tests`目录下新建一个测试命名为`index.test.ts`, 写基本测试，证明测试和程序是可以work的。

```ts
import { assert, equal } from "../deps.ts";
const { test } = Deno;

test("should work", () => {
  const universal = 42;
  equal(42, universal);
  assert(42 === universal);
});

```

### 第一次运行测试

```shell
❯ make test
deno test --allow-env --allow-net -L info
Check file:///xxxx/web-api-based-deno/.deno.test.ts
running 1 tests
test should work ... ok (6ms)

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out (6ms)
```

### 建立测试固件

将测试中用到的通用的测试信息存放在测试固件（testFixtures）中，可以在测试中复用，且可以简化代码。

```ts
// tests/testFixtures.ts
export const TEST_PORT = 9000
```

### health 接口

health 接口可以作为系统的健康检查的一个出口，在运维平台中非常实用。对于此接口，我们只需要返回一个状态`OK`即可。其他情况可忽略。那么对应的`Todo`应该如下：

> 当访问到系统的时候，应该返回系统的状态，且为OK。

所以，测试代码如下：

```ts
import {
  assertEquals,
  Application,
  Router,
} from "../../deps.ts";
import { getHealthInfo } from "../../src/controllers/health.ts";
import {TEST_PORT} from '../testFixtures.ts'

const { test } = Deno;

test("health check", async () => {
  const expectResponse = {
    success: true,
    data: "Ok",
  };

  const app = new Application();
  const router = new Router();
  const abortController = new AbortController();
  const { signal } = abortController;

  router.get("/health", async ({ response }) => {
    getHealthInfo({ response });
  });

  app.use(router.routes());

  app.listen({ port: TEST_PORT, signal });

  const response = await fetch(`http://127.0.0.1:${TEST_PORT}/health`);

  assertEquals(response.ok, true);
  const responseJSON = await response.json();

  assertEquals(responseJSON, expectResponse);
  abortController.abort();
});

```

> #### given
- 上面的代码中，首先声明了我们期望的数据结构，即`expectResponse`；
- 然后创建一个应用程序和一个路由，
- 再创建一个终止应用的控制器，且从中取到信号标识，
- 接着， 向路由中添加一个`health`路由及其handler；
- 然后将路由挂在到应用程序上；
- 监听应用程序端口，且传入应用程序信号；
> #### when
- 给启动的应用发一个get请求，请求路径为`/health`;
> #### then
- 根据fetch到的结果进行判定，看收到的`response`是不是和期望的一致， 且在最后终止上面的应用程序。
- 到此，如果运行测试肯定会发生错误，解决问题的也很简单，就是去实现`getHealthInfo` handler。

#### 实现 `getHealthInfo` handler

在src/controller下新建`health.ts`，并以最简单的方案实现上面期望的结果，如下：

```ts
// src/controllers/health.ts
import { Response, Status } from "../../deps.ts";
import IResponse from "./model/IResponse.ts";

export const getHealthInfo = ({ response }: { response: Response }) => {
  response.status = Status.OK;
  const res: IResponse = {
    success: true,
    data: "Ok",
  };
  response.body = res;
};

```

#### 运行测试

运行测试命令，测试通过;

```shell
❯ make test
deno test --allow-env --allow-net -L info
Check file://xxx/web-api-based-deno/.deno.test.ts
running 2 tests
test should work ... ok (6ms)
test health check ... ok (3ms)

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out (9ms)
```

至此，使用`TDD`完成第一个简单的`health`接口；但对外没有暴露接口，所以需要在`src`目录中实现一个对外暴露该接口的应用。

##### 新建`config.ts`， 做应用程序的配置管理文件

```ts
// src/config.ts
const env = Deno.env.toObject();
export const APP_HOST = env.APP_HOST || "127.0.0.1";
export const APP_PORT = parseInt(env.APP_PORT) || 8000;

export const API_VERSION = env.API_VERSION || "/api/v1";

```

配置文件中，记录了应用程序启动的默认host, 端口，及数据库相关的信息，最后记录了应用程序api的前缀。

在开始之前，需要在`deps.ts`中引入所需要的库；

```ts
export {
  Application,
  Router,
  Response,
  Status,
  Request,
  RouteParams,
  Context,
  RouterContext,
  helpers,
  send,
} from "https://deno.land/x/oak/mod.ts";
```

##### 新建路由 `router.ts`, 引入`Heath.ts`并绑定路由

```ts
// src/router.ts
import { Router } from "../deps.ts";
import { API_VERSION } from "./config.ts";
import { getHealthInfo } from "./controllers/health.ts";

const router = new Router();

router.prefix(API_VERSION);
router
  .get("/health", getHealthInfo)
export default router;

```

##### 新建`index.ts`, 建立应用程序

```ts
// src/index.ts
import { Application, send } from "../deps.ts";
import { APP_HOST, APP_PORT } from "./config.ts";
import router from "./router.ts";


export const listenToServer = async (app: Application) => {
  console.info(`Application started, and listen to ${APP_HOST}:${APP_PORT}`);
  await app.listen({
    hostname: APP_HOST,
    port: APP_PORT,
    secure: false,
  });
};

export function createApplication(): Promise<Application> {
  const app = new Application();
  app.use(router.routes());
  return Promise.resolve(app);
}

if (import.meta.main) {
  const app = await createApplication();
  await listenToServer(app);
}
```

##### 启动应用

如果是VSCode， 可以使用`F5`功能键，快速启动应用，在低版本的 VS Code(1.47.2一下) 中可以启动调试。也可以以下命令启动；

```
❯ make dev
deno run --allow-net --allow-env ./src/index.ts
数据库链接成功！
Application started, and listen to 127.0.0.1:8000
```

##### 调用接口测试结果

这里使用VS Code 的[`Rest Client`](https://marketplace.visualstudio.com/items?itemName=humao.rest-client)插件进行辅助测试。

###### 请求体

```
// _resources/httpClient/healthCheck.http
GET http://localhost:8000/api/v1/health HTTP/1.1
```

###### 结果

```
HTTP/1.1 200 OK
content-length: 28
x-response-time: 0ms
content-type: application/json; charset=utf-8

{
  "success": true,
  "data": "Ok"
}
```

至此，完成第一个接口，有 Oak 提供应用服务，经过了`Unit test`和 `RestClient`的测试。完成了开始的`Todo`。

### 用户create接口(addUser)

添加用户涉及到`Controller`, `Service` 和 `Repository`, 所以我们按照三步来实现该接口。

#### Controller

Controller 层是对外提供服务的，用户添加接口可以为系统添加用户，那么对应的`Todo`如下：

> * 输入用户名和密码，返回特定数据结构的用户信息
> * 参数必须输入，否则抛异常
> * 如果输入错误参数，则抛异常

在此过程中，我们需要用到[`mock`](https://github.com/udibo/mock)来mock 第三方依赖。

导入所需依赖，并新建`UserController.test.ts`， 测试如下：

```ts
// tests/controllers/UserController.test.ts
import {
  stub,
  Stub,
  assertEquals,
  v4,
  assertThrowsAsync,
  Application,
  Router,
} from "../../deps.ts";
import UserController from "../../src/controllers/UserController.ts";
import IResponse from "../../src/controllers/model/IResponse.ts";
import UserService from "../../src/services/UserService.ts";
import IUser, { User } from "../../src/entity/User.ts";
import InvalidedParamsException from "../../src/exception/InvalidedParamsException.ts";
import {TEST_PORT} from '../testFixtures.ts'

const { test } = Deno;


const userId = v4.generate();
const registrationDate = (new Date()).toISOString();

const mockedUser: User = {
  id: userId,
  username: "username",
  registrationDate,
  deleted: false,
};

test("#addUser should return added user when add user", async () => {
  const userService = new UserService();
  const queryAllStub: Stub<UserService> = stub(userService, "addUser");
  const expectResponse = {
    success: true,
    data: mockedUser,
  };
  queryAllStub.returns = [mockedUser];
  const userController = new UserController();
  userController.userService = userService;

  const app = new Application();
  const router = new Router();
  const abortController = new AbortController();
  const { signal } = abortController;

  router.post("/users", async (context) => {
    return await userController.addUser(context);
  });

  app.use(router.routes());

  app.listen({ port: TEST_PORT, signal });

  const response = await fetch(`http://127.0.0.1:${TEST_PORT}/users`, {
    method: "POST",
    body: "name=name&password=123",
    headers: {
      "Content-Type": "application/x-www-form-urlencoded",
    },
  });

  assertEquals(response.ok, true);
  const responseJSON = await response.json();

  assertEquals(responseJSON, expectResponse);
  abortController.abort();

  queryAllStub.restore();
});

test("#addUser should throw exception about no params given no params when add user", async () => {
  const userService = new UserService();
  const queryAllStub: Stub<UserService> = stub(userService, "addUser");
  queryAllStub.returns = [mockedUser];
  const userController = new UserController();
  userController.userService = userService;

  const app = new Application();
  const router = new Router();
  const abortController = new AbortController();
  const { signal } = abortController;

  router.post("/users", async (context) => {
    await assertThrowsAsync(
      async () => {
        await userController.addUser(context);
      },
      InvalidedParamsException,
      "should given params: name ...",
    );
    abortController.abort();
    queryAllStub.restore();
  });

  app.use(router.routes());

  app.listen({ port: TEST_PORT, signal });

  const response = await fetch(`http://127.0.0.1:${TEST_PORT}/users`, {
    method: "POST",
    body: "",
    headers: {
      "Content-Type": "application/x-www-form-urlencoded",
    },
  });
  await response.body!.cancel();
});

test("#addUser should throw exception about no correct params given wrong params when add user", async () => {
  const userService = new UserService();
  const queryAllStub: Stub<UserService> = stub(userService, "addUser");

  queryAllStub.returns = [mockedUser];
  const userController = new UserController();
  userController.userService = userService;

  const app = new Application();
  const router = new Router();
  const abortController = new AbortController();
  const { signal } = abortController;

  router.post("/users", async (context) => {
    await assertThrowsAsync(
      async () => {
        await userController.addUser(context);
      },
      InvalidedParamsException,
      "should given param name and password",
    );
    abortController.abort();
    queryAllStub.restore();
  });

  app.use(router.routes());

  app.listen({ port: TEST_PORT, signal });

  const response = await fetch(`http://127.0.0.1:${TEST_PORT}/users`, {
    method: "POST",
    body: "wrong=params",
    headers: {
      "Content-Type": "application/x-www-form-urlencoded",
    },
  });
  await response.body!.cancel();
```

controller 这一层需要调用service的服;作为service，对于controller是一个第三方服务，因此需要将service的方法mock，并以参数的形式传入controller;下面这段代码就是mock的应用；

```ts
  const userService = new UserService();
  const queryAllStub: Stub<UserService> = stub(userService, "addUser");
  const expectResponse = {
    success: true,
    data: mockedUser,
  };
  queryAllStub.returns = [mockedUser];
  const userController = new UserController();
  userController.userService = userService;
```

在此解释两个测试，第一个测试即`#addUser should return added user when add user`;
> ##### given
* mock `UserService`,给UserService的 `addUser`方法打桩，并返回特定的用户结构;
* 新建测试服务，并将 `UserController`注册给post接口 `/users`;

> ##### when

* 传入正确的form类型的参数，用`fetch`请求`http://127.0.0.1:9000/users`;

> ##### then

* 对获取到的结果进行判定，并中断测试应用，将打桩的方法恢复。

在此解释两个测试，第二个测试即`#addUser should throw exception about no params given no params when add user`; `given`和`when`与第一个测试的`given`和`when`查不多，只是`body`参数为空;最重要的不同点是这次的`then`是在`when`里面，因为抛异常会在`handler`上抛，所以，需要将`then`的判定放在`handler` 上。这里用到了`Deno`的`assertThrowsAsync`来捕获异常并判定异常。

> ##### given

* mock `UserService`,给UserService的 `addUser`方法打桩，并返回特定的用户结构;
* 新建测试服务，并将 `UserController`注册给post接口 `/users`;

> ##### when

* 给`body`传入空参数，用`fetch`请求`http://127.0.0.1:9000/users`;

> ##### then

* `then`部分处于`given`的路由处理handler中，对异常进行捕获并判定，接着中断测试应用，将打桩的方法恢复。

### Service

### Repository



## 乱中取整


## Reference

* [web-api-based-deno: https://github.com/guzhongren/web-api-based-deno](https://github.com/guzhongren/web-api-based-deno)
* [博客:https://guzhongren.github.io/](https://guzhongren.github.io/)
* [图床:https://sm.ms/](https://sm.ms/)
* [Denoland: https://deno.land/](https://deno.land/)
* [VS Code: https://code.visualstudio.com/](https://code.visualstudio.com/)
* [Docker: https://www.docker.com/](https://www.docker.com/)
* [Typescript: https://www.typescriptlang.org/](https://www.typescriptlang.org/)
* [Node: https://nodejs.org/](https://nodejs.org/)
* [mock: https://github.com/udibo/mock](https://github.com/udibo/mock)

## Hereby declared（特此申明）

本文仅代表个人观点，与[ThoughtWorks](https://www.thoughtworks.com/) 公司无任何关系。

----
![谷哥说-微信公众号](/images/wechat/扫码_搜索联合传播样式-标准色版.png)
