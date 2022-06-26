# TargetSource 对象
#Java #Spring 

该接口用于用于获取当前的被代理对象。Spring 在代理时并不是直接代理目标对象，而是代理 TargetSource 接口实例，通过 TargetSource 间接获取真实的被代理对象。

TargetSource 是 “静态” 的话，则它所返回的被代理对象则始终一致，从而允许 AOP 框架对其进行优化。另外 TargetSource 也可以动态返回被代理对象时（比如池化、热替换）等。

关系图如下：

![TargetSource 关系图](https://my-images-repo.oss-cn-hangzhou.aliyuncs.com/spring/TargetSource.png)