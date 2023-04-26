# Golang+Vue轻松构建Web应用

最近疫情在家，空闲时间比较多，整理下之前写的Golang项目[Weave](https://github.com/qingwave/weave)，补充了一些功能，加了前端实现。作为一个Web应用模板，也算是功能比较齐全了，现将开发过程中遇到的一些问题、项目特性总结下。

## 介绍
Weave是一个基于`Go+Vue`实现的Web应用模板，支持前后端，拥有完整的认证、存储、Restful API等功能。

后端基于Golang开发，主要特性如下：
- Restful API，通过`gin`实现，支持`swagger`
- MVC架构
- 支持Postgres存储，可以轻松替换为MySQL，使用`gorm`接入
- Redis缓存
- 基于`JWT`认证
- 服务优雅终止
- 请求限速
- Docker容器管理，`Websocket`支持
- RBAC认证，由`Casbin`支持
- 其他支持`Prometheus`监控、格式化日志、`PProf`等

前端基于`Vue`开发，使用`ElementPlus`组件库
- Vue3开发，使用组合式API
- 使用`vite`快速编译
- 支持`WebShell`，基于`xtermjs`
- 图表功能，基于`echarts`
- 支持`WindiCSS`，减少CSS编写

主要界面如下：
- 登录界面
![login](https://github.com/qingwave/weave/raw/master/document/img/login.png)
- Dashboard界面
![dashboard](https://github.com/qingwave/weave/raw/master/document/img/dashboard.png)
- 应用界面
![apps](https://github.com/qingwave/weave/raw/master/document/img/app.png)
- WebShell界面
![webshell](https://github.com/qingwave/weave/raw/master/document/img/webshell.png)

## 项目结构
项目组织如下：
```bash
├── Dockerfile
├── Makefile
├── README.md
├── bin
├── config # server配置
├── docs # swagger 生成文件
├── document # 文档
├── go.mod
├── go.sum
├── main.go # server入口
├── pkg # server业务代码
├── scripts # 脚本
├── static # 静态文件
└── web # 前端目录
```

### 后端结构
后端按照`MVC`架构实现，参考了社区一些最佳实践，具体如下：
```bash
├── pkg
│   ├── common # 通用包
│   ├── config # 配置相关
│   ├── container # 容器库
│   ├── controller # 控制器层，处理HTTP请求
│   ├── database # 数据库初始化，封装
│   ├── metrics # 监控相关
│   ├── middleware # http中间件
│   ├── model # 模型层
│   ├── repository # 存储层，数据持久化
│   ├── server # server入口，创建router
│   └── service # 逻辑层，处理业务
```

### 前端结构
前端实现`Vue3`实现，与一般Vue项目类似
```bash
web
├── README.md
├── index.html
├── node_modules
├── package-lock.json
├── package.json
├── public
│   └── favicon.ico
├── src # 所有代码位于src
│   ├── App.vue # Vue项目入口
│   ├── assets # 静态文件
│   ├── axios # http请求封装
│   ├── components # Vue组件
│   ├── main.js
│   ├── router # 路由
│   ├── utils # 工具包
│   └── views # 所有页面
└── vite.config.js # vite配置
```

## 一些细节

### 为什么使用JWT
主要是为了方便服务横向扩展，如果基于`Cookie+Session`，`Session`只能保存在服务端，无法进行负载均衡。另外通过api访问，jwt可以放在HTTP Header的`Bearer Token`中。

当使用Websocket时，不支持HTTP Header，由于认证统一在中间件中进行，可以通过简单通过`cookie`存储，也可以单独为Websocket配置认证。

JWT不支持取消，可以通过在redis存入黑名单实现。

### 缓存实现
加入了缓存便引入了数据一致性问题，经典的解决办法是先写数据库再写缓存（Cache-Aside模式），实现最终一致性，业务简单的项目可以使用这种方法。

那先写缓存行不行？如果同时有一个写请求一读请求，写请求会先删除缓存，读请求缓慢未命中会将DB中的旧数据载入，可能会造成数据不一致。先写数据库则不会有这样的问题，如果要实现先写缓存，可以使用双删的办法，即写前后分别操作一次缓存，这样处理逻辑会更复杂。如果不想侵入业务代码，可以通过监听Binlog来异步更新缓存。

### 请求限流
限流使用了`golang.org/x/time/rate`提供的令牌桶算法，以应对突发流量，可以对单个IP以及Server层面实现请求控制。

需要特别注意的是限流应当区别长连接与短连接，比如`Weave`中实现了容器`exec`接口，通过Websocket登录到容器，不应该影响其他正常请求。

### 从零开发前端
前端而言完全是毫无经验，选用了`Vue3`，主要是文档比较全面适合新手。UI基于了`ElementPlus`，目前还是Beta版本，使用过程了也遇到了一些Bug，生产过程中不建议用，无奈的是目前`Vue3`好像也没有比较成熟的UI库。

Vue文档以及示例很详细，上手也挺快。主要是CCS不熟悉，调整样式上花了不少功夫，后来引入了[WindiCSS](https://windicss.org/), 只编写了少量的样式，其他全部依赖WindiCSS实现。其他路由、请求、图表参考对应的文档实现起来也很容易。

搭建了一个比较完整的管理平台，自己还是挺满意的，后面会不断优化，加一些其他特性。

## 运行
后端本地运行，需要依赖Docker，Makefile文件只在Linux下有效，其他平台请自行尝试
1. 安装数据库postgres与redis，初始化库
```bash
make init
```
2. 本地运行
```bash
make run
```

前端使用`vite`编译
```bash
cd web
npm i
npm run dev
```

更多见[ReadMe](https://github.com/qingwave/weave#readme)

## 总结
本文总结了`Weave`的架构与特性，以及开发过程中遇到的一些问题，从零开始实现一个完整的前后端Web应用，其他功能后面会不断优化。

项目链接见
- https://github.com/qingwave/weave


> Explore more in [https://qingwave.github.io](https://qingwave.github.io)

