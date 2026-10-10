# micrboot Code Wiki

> 类 Spring MVC 的轻量级 HTTP 服务框架，基于 Netty 实现，适用于后端纯接口服务器（尤其是 APP 服务端），支持拦截器、多接口合并、JSON/String 视图渲染及文件上传下载。

---

## 目录

- [1. 项目概述](#1-项目概述)
- [2. 整体架构](#2-整体架构)
- [3. 目录结构](#3-目录结构)
- [4. 模块职责](#4-模块职责)
- [5. 关键类与函数说明](#5-关键类与函数说明)
  - [5.1 核心启动模块](#51-核心启动模块)
  - [5.2 HTTP 上下文模块](#52-http-上下文模块)
  - [5.3 MVC 调度模块](#53-mvc-调度模块)
  - [5.4 注解模块](#54-注解模块)
  - [5.5 拦截器模块](#55-拦截器模块)
  - [5.6 参数解析模块](#56-参数解析模块)
  - [5.7 视图解析模块](#57-视图解析模块)
  - [5.8 异常解析模块](#58-异常解析模块)
  - [5.9 工具模块](#59-工具模块)
- [6. 请求处理流程](#6-请求处理流程)
- [7. 依赖关系](#7-依赖关系)
- [8. 项目运行方式](#8-项目运行方式)
- [9. 扩展指南](#9-扩展指南)

---

## 1. 项目概述

micrboot 是一个**轻量级 HTTP 服务框架**，其设计目标与特点如下：

| 特性 | 说明 |
|------|------|
| 类 Spring MVC 架构 | 注解驱动（`@ApiCommand` / `@ApiMethod` / `@ApiParam`），熟悉 Spring 的开发者可快速上手 |
| Netty 网络框架 | 直接以 Netty 作为服务端，剔除了 Web 容器（Tomcat/Jetty），启动快、资源占用低 |
| 拦截器机制 | 类似 Spring MVC 的 `preDispatch` / `postHandler` / `afterHandle` 三段式拦截器，支持有序执行 |
| 多接口合并 | 通过 `ApiDispatcher.innerDoService()` 可在一个接口内部调度其他接口，实现批量合并 |
| 视图渲染 | 内置 `JsonViewResolver` 与 `StringViewResolver`，可扩展 |
| 文件上传下载 | 基于 Netty 的 `HttpPostRequestDecoder` 解析 multipart，支持 `FileUpload` 参数注入 |

**技术栈**：Java 7 + Netty 4.1 + Spring Framework 4.2.5（仅 IoC，不依赖 Spring MVC） + Jackson 1.9 + SLF4J/Logback。

---

## 2. 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                        Spring IoC 容器                        │
│  (扫描 @ApiCommand，注入 Server / ApiDispatcher / 拦截器等)    │
└──────────────────────────┬──────────────────────────────────┘
                           │ 启动
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                      Server (Netty 引导)                     │
│  Pipeline: IdleStateHandler → HttpServerCodec →              │
│            HttpObjectAggregator → HttpSimpleChannelHandle     │
└──────────────────────────┬──────────────────────────────────┘
                           │ FullHttpRequest
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              HttpSimpleChannelHandle (Netty Handler)         │
│  将 FullHttpRequest 转换为 HttpRequest / HttpResponse         │
└──────────────────────────┬──────────────────────────────────┘
                           │ doService(request, response)
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                     ApiDispatcher (核心调度)                  │
│                                                              │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │ 拦截器链      │  │ URL→Method   │  │ 参数解析器链        │  │
│  │ HandlerExec  │  │ 映射 cache   │  │ ParamResolver      │  │
│  │ uteChain     │  │ CommandMap   │  │                    │  │
│  └─────────────┘  └──────────────┘  └────────────────────┘  │
│                                                              │
│           ┌──────────────────────────────────┐              │
│           │  视图解析器链 ViewResolver         │              │
│           │  (Json / String)                 │              │
│           └──────────────────────────────────┘              │
│           ┌──────────────────────────────────┐              │
│           │  异常解析器链 ExceptionResolver    │              │
│           └──────────────────────────────────┘              │
└──────────────────────────┬──────────────────────────────────┘
                           │ 业务方法 (被 @ApiCommand/@ApiMethod 标注)
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    业务 ApiCommand Beans                      │
└─────────────────────────────────────────────────────────────┘
```

架构分为自上而下的四层：

1. **容器层**：Spring IoC 负责 Bean 装配与注解扫描。
2. **网络层**：`Server` + `HttpSimpleChannelHandle`，基于 Netty 完成 HTTP 收发。
3. **调度层**：`ApiDispatcher` 负责路由分发、拦截器编排、参数解析、视图渲染、异常处理。
4. **业务层**：开发者用 `@ApiCommand` / `@ApiMethod` 编写的接口。

---

## 3. 目录结构

```
src/main/java/com/zhuanglide/micrboot/
├── Server.java                         # Netty HTTP 服务启动入口
├── HttpSimpleChannelHandle.java        # Netty Channel Handler，桥接 Netty 与 MVC
├── common/
│   └── MediaType.java                  # 预留的媒体类型定义（当前为空占位）
├── http/
│   ├── HttpRequest.java                # HTTP 请求上下文封装
│   └── HttpResponse.java               # HTTP 响应封装
├── mvc/
│   ├── ApiDispatcher.java              # 核心调度器（路由 + 拦截器 + 解析器编排）
│   ├── ApiCommandMapping.java          # URL → Method 的映射信息持有者
│   ├── ApiMethodParam.java             # 方法参数元信息封装
│   ├── ModelAndView.java               # 视图名 + 结果数据模型
│   ├── HandlerExecuteChain.java        # 拦截器执行链
│   ├── DefaultStrategies.properties    # 默认策略类配置
│   ├── annotation/
│   │   ├── ApiCommand.java             # 类级命令注解（相当于 @Controller + basePath）
│   │   ├── ApiMethod.java              # 方法级路由注解（路径 + HTTP method）
│   │   └── ApiParam.java               # 参数级注解（名称/必填/默认值/正则校验）
│   ├── interceptor/
│   │   ├── ApiInterceptor.java         # 拦截器接口
│   │   ├── AbstractApiInterceptor.java # 拦截器抽象适配（默认实现）
│   │   └── AccessInterceptor.java      # 访问日志拦截器（记录耗时/参数/结果）
│   └── resolver/
│       ├── ApiMethodParamResolver.java # 参数解析器接口
│       ├── ViewResolver.java           # 视图解析器抽象类
│       ├── ExceptionResolver.java      # 异常解析器接口
│       ├── exception/
│       │   └── DefaultExceptionResolver.java  # 默认异常处理（记日志 + 500）
│       ├── param/
│       │   ├── ApiMethodParamBasicTypeResolver.java       # 基本类型参数解析
│       │   ├── ApiMethodParamCommandContextResolver.java  # HttpRequest 参数注入
│       │   └── ApiMethodParamWebResolver.java              # Web 对象参数(FileUpload/Cookie/Response)
│       └── view/
│           ├── JsonViewResolver.java    # JSON 视图解析器
│           └── StringViewResolver.java  # String 视图解析器
└── util/
    └── IMApiUtils.java                 # HTTP 参数/Cookie 解析、通配匹配工具
```

---

## 4. 模块职责

| 模块（包） | 职责 |
|-----------|------|
| `com.zhuanglide.micrboot` | 框架入口：Netty Server 引导与 Channel Handler |
| `com.zhuanglide.micrboot.common` | 公共定义（预留，`MediaType` 当前为空占位类） |
| `com.zhuanglide.micrboot.http` | HTTP 请求/响应的上下文封装，屏蔽 Netty 底层 API |
| `com.zhuanglide.micrboot.mvc` | MVC 核心：调度器、映射、参数模型、视图模型、拦截器链 |
| `com.zhuanglide.micrboot.mvc.annotation` | 注解定义，驱动接口注册与参数绑定 |
| `com.zhuanglide.micrboot.mvc.interceptor` | 拦截器体系（接口 + 抽象适配 + 内置访问日志） |
| `com.zhuanglide.micrboot.mvc.resolver` | 解析器体系：参数解析、视图解析、异常解析 |
| `com.zhuanglide.micrboot.util` | 工具类：HTTP 参数/Cookie 解析、通配匹配 |

---

## 5. 关键类与函数说明

### 5.1 核心启动模块

#### `Server`

**所在文件**：[Server.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/Server.java)

Netty HTTP 服务引导类，实现 `ApplicationContextAware` 与 `InitializingBean`，由 Spring 管理。

| 成员/方法 | 说明 |
|-----------|------|
| `port` | 监听端口 |
| `charset` | 默认编码（UTF-8） |
| `useEpoll` | 是否启用 Linux epoll（需 native 库可用） |
| `bossThreads` / `workThreads` | Netty boss/work 线程数，默认为 CPU 核数 ×2 / ×4 |
| `idleTimeOut` | 空闲超时（秒），默认 30 |
| `maxLength` | HTTP 报文最大长度，默认 64MB |
| `start()` | 启动服务：选择 Epoll/Nio → 构建 `ServerBootstrap` → 配置 Pipeline → `bind(port).sync()` |
| `start(int port)` | 设置端口并启动 |
| `initStrategies(ApplicationContext)` | 初始化 `HttpSimpleChannelHandle`：优先从容器获取 Bean，否则自动创建 |
| `afterPropertiesSet()` | Spring Bean 初始化回调，触发 `initStrategies` |
| `setApplicationContext(...)` | 注入 Spring 上下文 |

Pipeline 构成（`initChannel`）：
1. `IdleStateHandler` — 空闲检测，超时关闭连接
2. `HttpServerCodec` — HTTP 编解码
3. `HttpObjectAggregator` — 聚合 HTTP 报文（受 `maxLength` 限制）
4. `httpSimpleChannelHandle` — 业务处理

---

#### `HttpSimpleChannelHandle`

**所在文件**：[HttpSimpleChannelHandle.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/HttpSimpleChannelHandle.java)

`@Sharable` 的 Netty `SimpleChannelInboundHandler<FullHttpRequest>`，桥接 Netty 与 MVC 层。

| 方法 | 说明 |
|------|------|
| `channelRead0(ctx, fullRequest)` | 解码失败返回 400；否则构造 `HttpRequest`/`HttpResponse`，调用 `dispatcher.doService()`，最后将 `FullHttpResponse` 写回并关闭连接 |
| `userEventTriggered(ctx, evt)` | 触发空闲事件时关闭 Channel |
| `initApiMethodDispatcher(context)` | 获取/创建 `ApiDispatcher` 实例 |
| `afterPropertiesSet()` | 初始化策略（拿到 `ApiDispatcher`） |

---

### 5.2 HTTP 上下文模块

#### `HttpRequest`

**所在文件**：[HttpRequest.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/http/HttpRequest.java)

HTTP 请求上下文，`Serializable`，封装 Netty 的 `FullHttpRequest`。

| 成员/方法 | 说明 |
|-----------|------|
| `time` | 请求时间戳 |
| `address` | 客户端 IP（依次尝试 X-Forwarded-For / Proxy-Client-IP / WL-Proxy-Client-IP / Channel 远端地址） |
| `httpMethod` | GET/POST/UPDATE/DELETE |
| `requestUrl` | 去掉 query string 的 URI |
| `headers` | `HttpHeaders` |
| `requestParamsMap` | `Map<String, List<String>>` 参数表（GET + POST 合并） |
| `requestFiles` | `Map<String, FileUpload>` 上传文件 |
| `cookies` | `Map<String, Cookie>` |
| `channel` | Netty `Channel` |
| `attachment` | `Map<String,Object>` 附加属性（拦截器间传递数据，如耗时） |
| `getParameter(name)` | 取参数首个值 |
| `getParameters(name)` | 取参数全部值 |
| `addParameter(name, value)` | 追加参数（用于接口合并场景构造内部请求） |
| `getFileUpload(name)` | 取上传文件 |
| `getCookie(name)` | 取 Cookie |
| `clone()` | 基于同一 `FullHttpRequest`/`Channel` 克隆（用于多接口合并） |

构造时通过 `IMApiUtils.fillParamsMap()` 与 `IMApiUtils.fillCookies()` 完成参数与 Cookie 的解析。

---

#### `HttpResponse`

**所在文件**：[HttpResponse.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/http/HttpResponse.java)

HTTP 响应封装，持有一个 `FullHttpResponse`。

| 方法 | 说明 |
|------|------|
| `setStatus(HttpResponseStatus)` | 设置状态码 |
| `addHeader(name, value)` | 追加响应头 |
| `addCookie(Cookie)` | 通过 `ServerCookieEncoder` 写入 Set-Cookie |
| `setContent(String)` | 以 `charset` 写入文本内容 |
| `setContent(byte[])` | 写入字节内容 |
| `setFile(fileName, bytes)` | 设置下载响应头并写入字节（文件下载） |
| `getHttpResponse()` | 取底层 `FullHttpResponse` |

---

### 5.3 MVC 调度模块

#### `ApiDispatcher`

**所在文件**：[ApiDispatcher.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/mvc/ApiDispatcher.java)

框架核心调度器，实现 `ApplicationContextAware` 与 `InitializingBean`。

**核心字段**：

| 字段 | 说明 |
|------|------|
| `cacheCommandMap` | `Map<String, Map<RequestMethod, ApiCommandMapping>>` — URL → (HTTP方法 → 映射) |
| `apiInterceptors` | 拦截器列表（按 `Ordered` 排序） |
| `viewResolvers` | 视图解析器列表 |
| `apiMethodParamResolvers` | 参数解析器列表 |
| `exceptionResolvers` | 异常解析器列表 |
| `defaultStrategies` | 从 `DefaultStrategies.properties` 加载的默认策略类配置 |

**核心方法**：

| 方法 | 说明 |
|------|------|
| `dispatcher(HttpRequest)` | 根据 HTTP method + path 从 `cacheCommandMap` 查找 `ApiCommandMapping`；先精确匹配方法，再回退 `ALL` |
| `handler(mapping, request, response)` | 参数解析与方法反射调用：遍历参数，用 `ApiMethodParamResolver` 解析；处理 AOP 代理（JDK/CGLIB）；最终 `ReflectionUtils.invokeMethod` |
| `doService(request, response)` | 对外入口：构建 `HandlerExecuteChain` → `innerDoService` → `processDispatchResult` |
| `innerDoService(request, response, chain, withInterceptor)` | 内部调度：`applyPreDispatch` → `dispatcher` 路由 → `applyPostHandle` → `handler` 调用。`withInterceptor=false` 时跳过拦截器（多接口合并用） |
| `innerDoService(request, response)` | 无链、带拦截器的便捷重载（对外暴露，用于接口合并） |
| `processDispatchResult(chain, request, response, throwable)` | 异常处理 → `render` 视图 → `triggerAfterCompletion` |
| `render(result, request, response)` | 遍历 `viewResolvers`，首个 `resolve` 成功者执行 `render` |
| `initStrategies(context)` | 初始化拦截器、参数解析器、视图解析器、异常解析器，并 `loadApiCommand` |
| `loadApiCommand(context)` | 扫描 `@ApiCommand` Bean，解析其 `@ApiMethod` 方法，填充 `cacheCommandMap`；处理 AOP 代理目标类 |
| `getBasePath(path)` / `getFullUrl(path, url)` | 规范化 basePath 与拼接完整 URL |
| `initViewResolver/initApiMethodParamResolvers/initExceptionResolvers/initApiInterceptor` | 各解析器/拦截器初始化：先查容器 Bean，无则用 `getDefaultStrategies` 从 properties 加载 |
| `getDefaultStrategies(context, strategyInterface)` | 依据 `DefaultStrategies.properties` 反射创建默认策略实例 |

---

#### `ApiCommandMapping`

**所在文件**：[ApiCommandMapping.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/mvc/ApiCommandMapping.java)

URL 到方法的映射信息持有者（值对象）。

| 字段 | 说明 |
|------|------|
| `command` | 完整 URL |
| `bean` | 业务 Bean（可能是 AOP 代理） |
| `proxyTargetBean` | 代理目标 Bean |
| `method` | 反射 `Method` |
| `paramNames` | 参数名数组（由 `LocalVariableTableParameterNameDiscoverer` 获取） |
| `parameterTypes` | 参数类型数组 |
| `paramAnnotations` | 参数注解二维数组 |
| `extendFields` | `Map<Class,Object>` 扩展注入字段（按类型注入额外对象） |

---

#### `ApiMethodParam`

**所在文件**：[ApiMethodParam.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/mvc/ApiMethodParam.java)

方法单个参数的元信息封装，供参数解析器使用。

| 字段 | 说明 |
|------|------|
| `method` | 所属方法 |
| `paramName` | 参数名 |
| `paramAnnotations` | 参数注解 |
| `paramType` | 参数 `Type` |

---

#### `ModelAndView`

**所在文件**：[ModelAndView.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/mvc/ModelAndView.java)

视图名 + 结果数据的模型。

| 字段/方法 | 说明 |
|-----------|------|
| `viewName` | 视图名（如 `JSON_VIEW`），用于匹配 `ViewResolver` |
| `result` | 业务返回数据 |
| `setResult(Object)` | 设置结果 |

---

#### `HandlerExecuteChain`

**所在文件**：[HandlerExecuteChain.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/mvc/HandlerExecuteChain.java)

拦截器执行链，管理拦截器的有序调用与异常处理。

| 方法 | 说明 |
|------|------|
| `applyPreDispatch(request, response)` | 正序执行所有拦截器 `preDispatch`；任一返回 false 则触发已执行拦截器的 `afterHandle` 并中止 |
| `applyPostHandle(request, response)` | 正序执行 `postHandler`（可改参数） |
| `triggerAfterCompletion(request, response, e)` | **逆序**执行已执行拦截器的 `afterHandle`（可改返回值/处理异常），再触发异常解析 |
| `triggerException(request, response, ex)` | 遍历 `exceptionResolvers` 处理异常 |
| `setMapping(mapping)` / `setResult(result)` | 设置当前映射与执行结果 |

---

#### `DefaultStrategies.properties`

**所在文件**：[DefaultStrategies.properties](file:///workspace/src/main/java/com/zhuanglide/micrboot/mvc/DefaultStrategies.properties)

默认策略类配置，当容器中未显式定义对应 Bean 时，按此文件反射创建。

```properties
com.zhuanglide.micrboot.mvc.resolver.ViewResolver = com.zhuanglide.micrboot.mvc.resolver.view.JsonViewResolver
com.zhuanglide.micrboot.mvc.resolver.ExceptionResolver = com.zhuanglide.micrboot.mvc.resolver.exception.DefaultExceptionResolver
com.zhuanglide.micrboot.mvc.resolver.ApiMethodParamResolver = \
  com.zhuanglide.micrboot.mvc.resolver.param.ApiMethodParamBasicTypeResolver,\
  com.zhuanglide.micrboot.mvc.resolver.param.ApiMethodParamCommandContextResolver,\
  com.zhuanglide.micrboot.mvc.resolver.param.ApiMethodParamWebResolver
```

> 注：`ApiInterceptor` 无默认实现，需在 Spring 配置中显式声明（如 `AccessInterceptor`）。

---

### 5.4 注解模块

#### `@ApiCommand`

**所在文件**：[ApiCommand.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/mvc/annotation/ApiCommand.java)

- 作用目标：`ElementType.TYPE`（类）
- 元注解 `@Component`：使类被 Spring 扫描为 Bean
- 属性 `value()`：基础路径，默认 `/`

标注在类上表示一个命令控制器，其路径作为所有方法的 basePath。

---

#### `@ApiMethod`

**所在文件**：[ApiMethod.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/mvc/annotation/ApiMethod.java)

- 作用目标：`ElementType.METHOD, TYPE`
- 属性 `value()`：方法路径（必填）
- 属性 `method()`：HTTP 方法枚举，默认 `ALL`
- 内部枚举 `RequestMethod`：`ALL / GET / POST / UPDATE / DELETE`，含 `equals(String)` 忽略大小写比较

---

#### `@ApiParam`

**所在文件**：[ApiParam.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/mvc/annotation/ApiParam.java)

- 作用目标：`ElementType.PARAMETER`
- 属性：
  - `value()`：参数名，默认空串（空时使用编译期参数名）
  - `required()`：是否必填，默认 `true`
  - `defaultValue()`：默认值，默认空串
  - `validateRegx()`：正则校验表达式，默认空串（不校验）

---

### 5.5 拦截器模块

#### `ApiInterceptor`（接口）

**所在文件**：[ApiInterceptor.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/mvc/interceptor/ApiInterceptor.java)

继承 `Ordered`，定义三段拦截：

| 方法 | 时机 | 作用 |
|------|------|------|
| `preDispatch(request, response)` | URL 路由之前 | 可改请求/拦截（返回 false 中止） |
| `postHandler(mapping, request, response)` | 路由后、invoke 之前 | 可增删改参数 |
| `afterHandle(mapping, modelView, request, response, throwable)` | invoke 之后 | 可改返回值/处理异常 |

---

#### `AbstractApiInterceptor`

**所在文件**：[AbstractApiInterceptor.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/mvc/interceptor/AbstractApiInterceptor.java)

适配类，提供默认空实现（`preDispatch` 返回 `true`），含 `order` 字段与 setter。

---

#### `AccessInterceptor`

**所在文件**：[AccessInterceptor.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/mvc/interceptor/AccessInterceptor.java)

内置访问日志拦截器，`order = HIGHEST_PRECEDENCE`（最先执行）。

| 行为 | 说明 |
|------|------|
| `preDispatch` | 记录 `REQ_START_TIME` 到 attachment |
| `postHandler` | 拼接请求参数字符串存入 `REQ_SYS_PARAMS` |
| `afterHandle` | 计算耗时，输出访问日志：`clientIp | url | headers | params | result | cost` |
| `accessLoggerName` | 可配置独立 Logger 名称，便于分离访问日志 |

---

### 5.6 参数解析模块

#### `ApiMethodParamResolver`（接口）

**所在文件**：[ApiMethodParamResolver.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/mvc/resolver/ApiMethodParamResolver.java)

| 方法 | 说明 |
|------|------|
| `support(ApiMethodParam)` | 是否支持该参数 |
| `getParamObject(ApiMethodParam, request, response)` | 解析并返回参数值 |

解析器按列表顺序尝试，首个 `support` 返回 true 者生效。

---

#### `ApiMethodParamBasicTypeResolver`

**所在文件**：[ApiMethodParamBasicTypeResolver.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/mvc/resolver/param/ApiMethodParamBasicTypeResolver.java)

解析基本类型参数。

- 支持类型集合 `BASIC_TYPE_CLASS`：`String`、`BigInteger`、`Byte/byte`、`Short/short`、`Integer/int`、`Long/long`、`Float/float`、`Double/double`、`Character/char`、`Boolean/boolean`、`int[]`、`long[]`、`String[]`
- 解析逻辑：
  1. 读取 `@ApiParam` 的 `value`/`required`/`defaultValue`/`validateRegx`
  2. 从 `request.getParameter()` 取值；必填校验；默认值回退；正则校验
  3. 按类型转换（`Integer.parseInt`、`Long.parseLong` 等）
  4. 数组类型按 `arraySplit`（默认 `,`）切分
- 可通过 `setArraySplit` 配置分隔符

---

#### `ApiMethodParamCommandContextResolver`

**所在文件**：[ApiMethodParamCommandContextResolver.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/mvc/resolver/param/ApiMethodParamCommandContextResolver.java)

当参数类型为 `HttpRequest.class` 时，注入当前 `HttpRequest` 上下文。

> 注：与 `ApiMethodParamWebResolver` 对 `HttpRequest` 的支持存在重叠，实际生效顺序取决于 properties 中的排列顺序（BasicType → CommandContext → Web）。`HttpRequest` 不属于基本类型，故先由 CommandContext 解析命中。

---

#### `ApiMethodParamWebResolver`

**所在文件**：[ApiMethodParamWebResolver.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/mvc/resolver/param/ApiMethodParamWebResolver.java)

解析 Web 对象参数，支持类型：`HttpRequest`、`HttpResponse`、`FileUpload`、`Cookie`。

- `HttpRequest` → 注入 request
- `HttpResponse` → 注入 response
- `FileUpload` → `request.getFileUpload(paramName)`（文件上传）
- `Cookie` → `request.getCookie(paramName)`

---

### 5.7 视图解析模块

#### `ViewResolver`（抽象类）

**所在文件**：[ViewResolver.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/mvc/resolver/ViewResolver.java)

继承 `Ordered`，含 `order` 与 `contentType`。

| 方法 | 说明 |
|------|------|
| `resolve(Object result)` | 判断是否支持渲染该结果，返回 `ModelAndView` 或 null |
| `render(ModelAndView, request, response)` | 渲染到 response |

---

#### `JsonViewResolver`

**所在文件**：[JsonViewResolver.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/mvc/resolver/view/JsonViewResolver.java)

- `contentType = "application/json"`
- `viewName = "JSON_VIEW"`（默认常量 `DEFAULT_JSON_VIEW_NAME`）
- `resolve`：仅当结果为 `ModelAndView` 且 `viewName` 匹配时返回
- `render`：用 Jackson `ObjectMapper.writeValueAsString()` 序列化 `result` 写入 response
- `afterPropertiesSet`：若未注入 `ObjectMapper` 则新建

---

#### `StringViewResolver`

**所在文件**：[StringViewResolver.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/mvc/resolver/view/StringViewResolver.java)

- `contentType = "text/html"`
- `resolve`：结果为 `ModelAndView` 且 `result` 是 `String` 时支持；或结果直接是 `String` 时包装成 `ModelAndView`
- `render`：`response.setContent((String) mv.getResult())`

---

### 5.8 异常解析模块

#### `ExceptionResolver`（接口）

**所在文件**：[ExceptionResolver.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/mvc/resolver/ExceptionResolver.java)

继承 `Ordered`，单方法 `resolveException(mapping, request, response, Throwable)`。

---

#### `DefaultExceptionResolver`

**所在文件**：[DefaultExceptionResolver.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/mvc/resolver/exception/DefaultExceptionResolver.java)

- `order = LOWEST_PRECEDENCE`
- 行为：记录错误日志，设置响应状态 `500 INTERNAL_SERVER_ERROR`

---

### 5.9 工具模块

#### `IMApiUtils`

**所在文件**：[IMApiUtils.java](file:///workspace/src/main/java/com/zhuanglide/micrboot/util/IMApiUtils.java)

| 方法 | 说明 |
|------|------|
| `fillParamsMap(FullHttpRequest, HttpRequest)` | 解析 URI query 与 POST body（含 multipart 文件），填充 `requestParamsMap` 与 `requestFiles` |
| `fillCookies(FullHttpRequest, HttpRequest)` | 从 `Cookie` 头用 `ServerCookieDecoder` 解析，填充 `cookies` |
| `leftMatch(src, wildcard)` | 左匹配：`*` 结尾表示前缀匹配，否则全等比较（用于通配路由/方法名匹配） |

---

## 6. 请求处理流程

以 `GET /t1?name=hello&age=18` 为例，完整调用链如下：

```
客户端请求
   │
   ▼
Server (Netty bind)
   │  Pipeline 处理
   ▼
HttpSimpleChannelHandle.channelRead0(ctx, FullHttpRequest)
   │  ① 解码校验，失败返回 400
   │  ② 构造 HttpRequest（自动解析 params/cookies）
   │  ③ 构造 HttpResponse
   ▼
ApiDispatcher.doService(request, response)
   │  ④ new HandlerExecuteChain(interceptors, exceptionResolvers)
   ▼
innerDoService(request, response, chain, withInterceptor=true)
   │
   ├─ chain.applyPreDispatch()        【拦截器 preDispatch，正序，可中止】
   │     └─ AccessInterceptor.preDispatch: 记录开始时间
   │
   ├─ dispatcher(request)              【URL + HTTP method → ApiCommandMapping】
   │     └─ 查 cacheCommandMap，先精确方法后回退 ALL
   │     └─ 找不到则 response.setStatus(404) 返回
   │
   ├─ chain.applyPostHandle()          【拦截器 postHandler，正序，可改参数】
   │     └─ AccessInterceptor.postHandler: 拼接参数
   │
   ├─ handler(mapping, request, response)   【反射调用业务方法】
   │     └─ 遍历参数 → ApiMethodParamResolver.resolve → 解析参数值
   │        ├─ BasicTypeResolver: 解析 name/age 等
   │        ├─ CommandContextResolver: 注入 HttpRequest
   │        └─ WebResolver: 注入 FileUpload/Cookie/HttpResponse
   │     └─ AopUtils 判断代理 → ReflectionUtils.invokeMethod → 返回 ModelAndView
   │
   ▼
processDispatchResult(chain, request, response, throwable)
   │  ⑤ 若有异常 → chain.triggerException() → ExceptionResolver
   │  ⑥ render(result) → ViewResolver.resolve/render
   │     └─ JsonViewResolver: 序列化 result 为 JSON 写入 response
   │  ⑦ chain.triggerAfterCompletion() 【拦截器 afterHandle，逆序】
   │     └─ AccessInterceptor.afterHandle: 输出访问日志
   ▼
HttpSimpleChannelHandle 写回 FullHttpResponse → 关闭连接
```

**多接口合并**场景通过 `ApiDispatcher.innerDoService(request, response)`（无链重载）在业务方法内主动调度其他接口，可控制是否启用拦截器。

---

## 7. 依赖关系

### 7.1 Maven 依赖（pom.xml）

| 依赖 | 版本 | Scope | 用途 |
|------|------|-------|------|
| `io.netty:netty-all` | 4.1.0.Final | compile | HTTP 服务端核心 |
| `org.springframework:spring-context` | 4.2.5.RELEASE | provided | IoC 容器 |
| `org.springframework:spring-beans` | 4.2.5.RELEASE | provided | Bean 工厂 |
| `org.springframework:spring-core` | 4.2.5.RELEASE | provided | 工具类（ReflectionUtils/ClassUtils 等） |
| `org.slf4j:slf4j-api` | 1.7.21 | provided | 日志门面 |
| `commons-fileupload:commons-fileupload` | 1.3.1 | provided | 文件上传（实际由 Netty multipart 解析，此为兼容） |
| `commons-httpclient:commons-httpclient` | 3.1 | provided | HTTP 客户端（工具备用） |
| `org.codehaus.jackson:jackson-mapper-asl` | 1.9.13 | provided | JSON 序列化（JsonViewResolver） |
| `org.springframework:spring-test` | 4.2.5.RELEASE | test | 测试支持 |
| `junit:junit` | 4.12 | test | 单元测试 |
| `ch.qos.logback:logback-core` | 1.1.3 | test | 日志实现 |
| `ch.qos.logback:logback-classic` | 1.1.3 | test | 日志实现 |

> 多数核心依赖为 `provided`：框架本身不打包这些库，由使用方的工程提供具体版本，避免版本冲突。`netty-all` 为 `compile`，是唯一强依赖。

### 7.2 模块间依赖

```
Server ──依赖──► HttpSimpleChannelHandle ──依赖──► ApiDispatcher
                                                      │
                 ┌────────────────────────────────────┤
                 ▼                  ▼                 ▼
          HandlerExecuteChain   ApiCommandMapping   ModelAndView
                 │                  │
                 ▼                  ▼
          ApiInterceptor       @ApiCommand/@ApiMethod/@ApiParam
                 │
                 ▼
          ExceptionResolver ◄── DefaultExceptionResolver

ApiDispatcher ──使用──► ApiMethodParamResolver ──实现──► BasicType/CommandContext/Web Resolver
                ──使用──► ViewResolver ──实现──► JsonViewResolver / StringViewResolver
                ──使用──► ExceptionResolver

HttpRequest ──使用──► IMApiUtils（解析 params/cookies）
```

---

## 8. 项目运行方式

### 8.1 环境要求

- JDK 1.7+（pom 配置 source/target 1.7）
- Maven 3.x
- Spring Framework 4.2.5（provided，由使用方引入）

### 8.2 编译打包

```bash
mvn clean package
```

产物：`target/com.zhuanglide.micrboot-1.0-SNAPSHOT.jar`

### 8.3 集成使用

**第 1 步：Spring 配置文件**（如 `api.xml`）

```xml
<!-- 扫描 @ApiCommand 注解类 -->
<context:component-scan base-package="com.yourpkg.**">
    <context:include-filter type="annotation"
            expression="com.zhuanglide.micrboot.mvc.annotation.ApiCommand"/>
</context:component-scan>

<!-- 配置并启动 Server -->
<bean name="server" class="com.zhuanglide.micrboot.Server">
    <property name="port" value="8080"/>
    <!-- 可选配置 -->
    <property name="charset" value="UTF-8"/>
    <property name="threadNum" value="16"/>
    <property name="bossThreads" value="16"/>
    <property name="workThreads" value="32"/>
    <property name="idleTimeOut" value="30"/>
    <property name="maxLength" value="67108864"/>
    <property name="useEpoll" value="false"/>
</bean>
```

**第 2 步：编写业务接口**

```java
@ApiCommand
public class ApiCommandTest {
    @ApiMethod("/t1")
    public ModelAndView test1(@ApiParam("name") String name,
                             @ApiParam("age") int age) {
        ModelAndView mv = new ModelAndView("JSON_VIEW");
        Map<String, Object> res = new HashMap<>();
        res.put("name", name);
        res.put("age", age);
        mv.setResult(res);
        return mv;
    }
}
```

**第 3 步：启动服务**

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"classpath*:api.xml"})
public class TestServer {
    @Autowired
    private Server server;

    @Test
    public void main() {
        server.start();
    }
}
```

**第 4 步：调用验证**

```
GET http://localhost:8080/t1?name=hello&age=18
```

响应：

```json
{
    "name": "hello",
    "age": 18
}
```

### 8.4 启用拦截器

在 Spring 配置中声明拦截器 Bean 即可被自动加载：

```xml
<bean class="com.zhuanglide.micrboot.mvc.interceptor.AccessInterceptor">
    <property name="order" value="0"/>
    <property name="accessLoggerName" value="access-log"/>
</bean>
```

自定义拦截器继承 `AbstractApiInterceptor` 并实现 `getOrder`/`setOrder`。

### 8.5 启用多接口合并

```xml
<bean class="com.zhuanglide.micrboot.mvc.ApiDispatcher"/>
```

在业务方法中注入 `ApiDispatcher`，通过 `innerDoService` 调度其他接口（参见 README 示例）。

---

## 9. 扩展指南

| 扩展点 | 接口/抽象类 | 实现方式 |
|--------|-----------|---------|
| 自定义拦截器 | `ApiInterceptor` / `AbstractApiInterceptor` | 实现 `preDispatch`/`postHandler`/`afterHandle`，注册为 Spring Bean |
| 自定义参数解析 | `ApiMethodParamResolver` | 实现 `support` + `getParamObject`，注册为 Bean 或加入 `DefaultStrategies.properties` |
| 自定义视图 | `ViewResolver` | 继承抽象类实现 `resolve` + `render`，注册为 Bean |
| 自定义异常处理 | `ExceptionResolver` | 实现 `resolveException`，注册为 Bean |
| 自定义 Dispatcher | — | 替换 `ApiDispatcher`（通过 `DefaultStrategies.properties` 的 Dispatcher 键，当前未在 properties 中启用） |

**策略加载优先级**：Spring 容器中显式定义的 Bean > `DefaultStrategies.properties` 默认实现。
