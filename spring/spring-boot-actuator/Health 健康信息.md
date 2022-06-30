# Health 健康信息
#Java #Spring 

在应用程序中可以通过健康信息来检查正在运行的状态。当生存系统出现故障时，监控软件经常使用它来发出提醒通知。

健康端点公开公开的信息取决于 `management.endpoint.health.show-details` 和 `management.endpoint.health.show-components` 属性，并通过以下值进行配置：

+ never：从不显示细节。
+ when-authorized：信息仅向授权用户显示。
+ always：向所有用户显示详细信息。

默认情况下，值为 never。当用户位于多个端点的角色时，则被认为是被授权的。如果端点没有配置角色，则所有经过身份验证的用户都被认为是授权的。可以使用 `management.endpoint.health.roles` 属性配置角色。

## 信息来源

健康信息是从 HealthContributorRegistry 中收集的，默认情况下使用 ApplicationContext 中定义的 HealthContributor 实例。Spring Boot 中也包含许多自动配置的 HealthContributor，同时也支持自行编写。

HealthContributor 存在两个子接口：

+ HealthIndicator：提供实际的健康信息，其中包括状态。
+ CompositeHealthContributor：提供其他 HealthContributor 组合。

总之，它们会形成一个树结构来表示整个系统的健康状况。

默认情况下，最终的系统运行状况由 StatusAggregator 派生，它根据状态的有序列表来自对每个 HealthIndicator 状态的排序。排序列表中的第一个状态用作整体健康状态，如果没有 HealthIndicator 返回 StatusAggregator 已知状态，则使用 UNKNOWN 状态。

> 在运行时，可以通过 HealthContributorRegistry 注册或者取消注册健康指标。

## 自动配置的健康指标

在适当的时候，Spring Boot 会自动配置部分 HealthIndicators，可以通过配置 `management.health.key.enabled` 来启用或者禁用选定的指标：

| key           | 名称                             | 描述                               |
| ------------- | -------------------------------- | ---------------------------------- |
| db            | DataSourceHealthIndicator        | 检查是否可以获取 DataSource 的连接 |
| diskspace     | DiskSpaceHealthIndicator         | 检查磁盘空间不足                   |
| elasticsearch | ElasticsearchRestHealthIndicator | 检查 Elasticsearch 集群是否启动    |
| jms           | JmsHealthIndicator               | 检查 JMS 代理是否已启动            |
| mail          | MailHealthIndicator              | 检查邮件服务器是否已启动           |
| mongo         | MongoHealthIndicator             | 检查 Mongo 数据库是否已启动        |
| ping          | PingHealthIndicator              | 始终以 UP 响应                     |
| redis         | RedisHealthIndicator             | 检查 Redis 服务器是否已启动        | 

> 可以通过设置 `management.health.defaults.enabled` 属性来禁用它们。

## 自定义健康指标

如果要提供自定义健康信息，可以注册实现 HealthIndicator 接口的 Spring Bean。实现接口的 health 方法并返回 Health 响应，并且可以选择包含要显示的其他详细信息。

例如：

```java
@Component
public class MyHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        int errorCode = check();
        if (errorCode != 0) {
            return Health.down().withDetail("Error Code", errorCode).build();
        }
        return Health.up().build();
    }

    private int check() {
        // perform some specific health check
        return ...
    }

}
```

给定 HealthIndicator 实例的对应 Bean 名称是不携带 HealthIndicator 后缀。例如，上述例子中 Bean 名称为 my。

> 健康指标通常通过 HTTP 调用，并且需要在连接超时前做出响应。因此当响应时间超过 10 秒的健康指标，Spring Boot 都会记录一条警告信息。如果要配置该值，可使用 `management.endpoint.health.logging.slow-indicator-threshold` 属性。

除了 Spring Boot 的预定义状态类型之外，Health 还可以返回一个表示新系统状态的自定义状态。这种情况下，需要提供 StatusAggregator 接口自定义实现，或者必须使用 `management.endpoint.health.status.order` 配置属性来配置默认实现。

例如，假设你的 HealthIndicator 实现中实现了代码为 FATAL 的新状态。要配置严重性顺序，可以将以下属性添加到程序中：

```properties
management.endpoint.health.status.order=fatal,down,out-of-service,unknown,up
```

另外，响应中的 HTTP 状态码也反映了整体健康状况。默认情况下，OUT_OF_SERVICE 和 DOWN 映射到 503 上，任何未映射的健康状态，包括 UP 都映射到 200。如果通过 HTTP 访问健康端点，可能还需要注册自定义状态映射。

配置自定义映射会禁用 DOWN 和 OUT_OF_SERVICE 的默认映射。如果要保留默认映射，则必须显式配置它们以及任何自定义映射。例如，以下属性将 FATAL 映射到 503 并保留 DOWN 和 OUT_OF_SERVICE 的默认映射：

```properties
management.endpoint.health.status.http-mapping.down=503
management.endpoint.health.status.http-mapping.fatal=503
management.endpoint.health.status.http-mapping.out-of-service=503
```

如果需要更多控制，可以自定义 HttpCodeStatusMapper 实例 Bean。
