# Environment 环境对象
#Java #Spring 

用于表示当前程序的运行环境，Environment 继承于 PropertyResolver 接口。PropertyResolver 接口用于解析属性，而 Environment 在此基础上进行了扩展，支持配置文件的访问。

属性可能有多个来源，例如属性文件、JVM 系统属性、系统环境变量、Servlet 上下文等。Environment 的作用就是提供一个统一的访问服务接口，用于获取属性。

配置文件是一个具有名称的属性组，只有在配置文件活跃时，对应的属性才会生效；配置文件也可以是一个具有名称 Bean 定义组，通过 @Profile 注解标记只有当指定配置文件活跃时，才会向容器中注册 Bean 定义中对应的 Bean。

## 获取 Environment 对象

在程序中如果需要使用到 Environment 时，则可以在需要使用的类上实现 EnvironmentAware 接口，容器在注册该类的 Bean 时会自动回调接口对应的方法传入 Environment 对象。

## ConfigurableEnvironment

实际上 Environment 接口只提供了关于配置文件和属性的读取接口，而配置文件对应的设置接口则定义在 Environment 的子接口 ConfigurableEnvironment 中。

同时 ConfigurableEnvironment 可获取对应的属性集合进行添加、删除等存在，例如调用 getPropertySources 方法获取 MutablePropertySources。

