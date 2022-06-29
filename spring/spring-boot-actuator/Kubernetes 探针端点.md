# Kubernetes 探针端点
#Java #Spring #K8s

通过 K8s 部署的应用程序可以通过 Container Probes 提供有关内部状态的信息。根据所提供的 K8s 配置信息，kubelet 会调用探测器并对结果做出反应。

默认情况下，Spring Boot 管理应用程序可用性状态。如果部署在 K8s 环境中，Actuator 会从 ApplicationAvailability 接口中收集 “Liveness” 和 “Readiness” 信息，并将信息用于专用健康指标：`LivenessStateHealthIndicator` 和 `ReadinessStateHealthIndicator`。

同时这些信息也会显示在全局健康端点上，还通过使用健康组作为单独的 HTTP 探测器公开：

+ /actuator/health/liveness
+ /actuator/health/readiness

之后可以在 K8s 配置中进行设置到：

```yaml
livenessProbe:
  httpGet:
    path: "/actuator/health/liveness"
	port: <actuator-port>
  failureThreshold: ...
  periodSeconds: ...
  
readinessProbe:
  httpGet:
    path: "/actuator/health/readiness"
	port: <actuator-port>
  failureThreshold: ...
  periodSeconds: ...
```

Spring Boot 相关的配置需要通过以下配置开启：

```properties
management.endpoint.health.probes.enabled=true
```