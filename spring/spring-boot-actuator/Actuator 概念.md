# Actuator 概念

actuator 是一个制造术语，指用于移动或控制某物的机械装置。actuator 可以通过微小的变化产生大量的运动。

spring-boot-actuator 包含许多附加功能，可以在应用程序部署到生产环境时帮助对其进行监控和管理。这个过程中可以选择使用 HTTP 或者 JMX 来管理和监视应用程序。


## Endpoints

端点使我们可以监视应用程序并与之交互，类似于 Api 接口。Spring Boot 包含许多内置端点，也可以自行添加自定义端点。

可以启用或禁用每个单独的端点并通过 HTTP 或者 JMX 访问，只有当端点被启用和公开时，它才被认为是可用的。

> 内置的端点只有在可用时，才会被自动配置。

### 通用端点

| ID             | 描述                                                                                                          |
| -------------- | ------------------------------------------------------------------------------------------------------------- |
| auditevents    | 公开当前应用程序的审计事件信息，但需要 AuditEventRepository 类型 Bean。                                       |
| beans          | 显示应用程序中所有的 bean 列表。                                                                              |
| caches         | 公开可用的缓存。                                                                                              |
| conditions     | 显示在配置和自动配置类上评估的条件以及匹配或不匹配的原因。                                                    |
| configprops    | 显示所有 @ConfigurationProperties 的列表。                                                                    |
| env            | 在 ConfigurableEnvironment 中公开的属性。                                                                     |
| health         | 显示应用程序运行状况信息。                                                                                    |
| httptrace      | 显示 HTTP 跟踪信息（默认最近 100 个 HTTP 请求 - 响应），但需要 HttpTraceRepository 类型 Bean。                |
| info           | 显示任意应用程序信息。                                                                                        |
| loggers        | 显示和修改应用程序中 logger 的配置。                                                                          |
| metrics        | 显示应用程序的指标信息。                                                                                      |
| mappings       | 显示所有 @RequestMapping 的路径列表。                                                                         |
| scheduledtasks | 显示应用程序中的计划任务。                                                                                    |
| quartz         | 显示所有关于 Quartz 调度程序作业信息。                                                                        |
| shutdown       | 可以让应用程序正常关闭，默认关闭。                                                                            |
| startup        | 显示 ApplicationStartup 收集到的启动步骤数据，但需要在 SpringApplication 中配置 BufferingApplicationStartup。 |
| threaddump     | 执行线程 dump 信息。                                                                                          |
 
### Web 端点

| ID         | 描述                                                                                                                           |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------ |
| heapdump   | 返回堆转储文件。在 HotSpot VM 上，返回 HPROF 格式文件；在 OpenJ9 VM 上，返回 PHD 格式文件。                                    |
| logfile    | 返回日志文件内容（需要设置 `logging.file.name` 或 `logging.file.path` 属性）。可支持使用 HTTP Range 头部获取部分日志文件内容。 |
| prometheus | 返回 Prometheus 可以抓取的格式，但需要引入 micrometer-registry-prometheus 依赖。                                               | 

### 启用端点

默认情况下，除了 shutdown 外的所有端点都启动。要配置端点的启动，则需要配置 `management.endpoint.<id>.enabled` 属性：

```properties
management.endpoint.shutdown.enabled=true
```

如果需要默认情况下所有的端点都禁用，则需要如下设置：

```properties
management.endpoints.enabled-by-default=false
```

该属性设置之后，所有的端点都需要显式开始。

## 暴露端点

被禁用的端点将完全从应用程序上下文中删除。但如果只是想更改暴露端点的方式，可以改为用 include 或 exclude 属性。

内置端点默认暴露配置：

| ID             | JMX | Web |
| -------------- | --- | --- |
| auditevents    | Yes | No  |
| beans          | Yes | No  |
| caches         | Yes | No  |
| conditions     | Yes | No  |
| configprops    | Yes | No  |
| env            | Yes | No  |
| health         | Yes | Yes |
| heapdump       | N/A | No  |
| httptrace      | Yes | No  |
| info           | Yes | No  |
| logfile        | N/A | No  |
| loggers        | Yes | No  |
| metrics        | Yes | No  |
| mappings       | Yes | No  |
| prometheus     | N/A | No  |
| quartz         | Yes | No  |
| scheduledtasks | Yes | No  |
| shutdown       | Yes | No  |
| startup        | Yes | No  |
| threaddump     | Yes | No  | 

当需要更改暴露端点的方式时，则可以使用特定技术包含和排除属性：

| 属性                                      | 默认值 |
| ----------------------------------------- | ------ |
| management.endpoints.jmx.exposure.exclude |        |
| management.endpoints.jmx.exposure.include | *      |
| management.endpoints.web.exposure.exclude |        |
| management.endpoints.web.exposure.include | health | 

include 属性列出公开的端点 ID，exclude 属性列出了不应公开的端点 ID。如果端点 ID 同时定义在 include 和 exclude 属性中时，exclude 属性优于 include 属性。

> 当需要指定所有端点时，则可以使用 `*` 代替。

## 配置端点

端点可以对不带任何参数的读取操作的响应结果进行缓存，要配置缓存的响应时间，请使用 `cache.time-to-live` 属性。

例如，将 beans 端点缓存信息的生存时间设置为 10 秒：

```properties
management.endpoint.beans.cache.time-to-live=10s
```

## 端点发现页面

默认情况下，端点存在一个 “发现页面”，路径为 `/actuator`，其中包含指向所有端点的链接。若要更改 “发现页面” 地址，则可以如下设置：

```properties
management.endpoints.web.base-path=/management
```

如果需要禁用 “发现页面”，则进行以下设置：

```properties
management.endpoints.web.discovery.enabled=false
```

## CORS 支持

跨域资源共享是一种 W3C 规范，可以以灵活的方式指定授权哪种跨域请求。在 Spring MVC 中 CORS 支持默认禁用，只有在设置 `management.endpoints.web.cors.allowed-origins` 属性后才会被启用。

例如，允许来自 example.com 的 GET 和 POST 调用：

```properties
management.endpoints.web.cors.allowed-origins=https://example.com
management.endpoints.web.cors.allowed-methods=GET,POST
```






