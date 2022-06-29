# 自定义 Endpoint 端点
#Java #Spring 

## 自定义 Web 端点

当 @Bean 注解伴随着 @Endpoint 注解时，所注册的 Bean 中携带 @ReadOperation、@WriteOperation 或者 @DeleteOperation 注解的方法都会自动通过 JMX 公开，并且也可以通过 HTTP 公开。

如果只是想通过单一的方式进行公开，则可以使用 @JmxEndpoint 或者 @WebEndpoint 编写特定方式的端点：

+ @JmxEndpoint：端点仅通过 JMX 进行公开。
+ @WebEndpoint：端点仅通过 HTTP 进行公开。

除此之外，还可以使用 @EndpointWebExtension 或者 @EndpointJmxExtension 对特定方式下的端点进行扩展。

### 接收输入

端点上的操作通过参数进行接收输入。当通过 Web 公开时，参数的取值通过 URL 的请求参数和 JSON 格式请求正文来的；而通过 JMX 公开时，参数将映射到 MBean 操作的参数，且默认情况下参数是必须的，但可以通过 @javax.annotation.Nullable 或者 @org.springframework.lang.Nullable 注解使其成为可选的。

例如，将携带 name 和 counter 的 JSON 请求正文映射到端点的参数，JSON 请求正文如下：

```json
{
	"name": "test",
	"counter": 42
}
```

对应的操作如下：

```java
@WriteOperation
public void updateData(String name, int counter) {
    // injects "test" and 42
}
```

注意，端点只能在方法参数中指定简单类型，无法支持复杂属性的类型声明。

> 为了让输入映射到方法参数，实现端点的 Java 代码应该使用 `-java-parameters` 参数编译。如果使用 Maven 的 spring-boot-starter-parent 插件，则会自动进行。

### 输入类型转换

如有必要，传递给端点操作方法的参数会自动转换为所需的类型。

在调用方法前，通过 JMX 或者 HTTP 接收的输入通过 ApplicationConversionService 实例以及使用 @EndpointConverter 指定的 Converter 或者 GenericConverter 相关 Bean 转换为所需的类型。

### Web 端点路径定义

端点的路径由基本路径和端点的 ID 组成。默认情况下，基本路径是 /actuator。

另外，还可以通过 @Selector 注释方法中的一个或者多个参数来进一步自定义路径，这样的参数作为路径变量添加到路径中。

当调用端点操作时，变量的值被传递到操作方法中。如果要捕获所有剩余的路径元素，可以使用 @Selector(Match=ALL_REMAINING) 添加到最后一个参数中，并转换成 String[] 可以兼容的类型。

### Web 端点方式定义

端点的请求方式由操作类型决定：

| 操作             | 方式   |
| ---------------- | ------ |
| @ReadOperation   | GET    |
| @WriteOperation  | POST   |
| @DeleteOperation | DELETE | 

### Web 端点 consumes 定义

对应 @WriteOperation 注释方法的请求体，请求 consumes 必须为 `application/vnd.spring-boot.actuator.v2+json` 或者 `application/json` 类型。而其他操作，consumes 子句必须为空。

### Web 端点 produces 定义

使用 @DeleteOperation、@ReadOperation 或者 @WriteOperation 注释时，如果方法返回 void 或者 Void，则 produces 为空；如果方法返回 org.springframework.core.io.Resource，则 produces 子句为 `application/octet-stream`。对于所有其他操作，produce 子句是 `application/vnd.spring-boot.actuator.v2+json` 或者 `application/json`。

### Web 端点响应状态

端点操作的响应状态取决于操作类型以及返回内容：

+ 如果 @ReadOperation 存在返回值，则响应 200；如果没有返回值，则响应 404。
+ 如果 @WriteOperation 或者 @DeleteOperation 存在返回值，则响应 200；如果没有返回值，则响应 204。
+ 如果调用方法时不带必填参数或者带无法转换的参数时，则不会调用方法，且响应状态为 400。

## 自定义 Controller 端点

可以使用 @ControllerEndpoint 或者 @RestControllerEndpoint 来实现仅由 Spring MVC 公开的端点。

通过使用 Spring MVC 的标准注解来映射方法，例如 @RequestMapping。端点的 ID 作为路径的前缀。Controller 端点提供与 Spring Web 框架更深层次的集成，但却以牺牲可移植性为代价。
