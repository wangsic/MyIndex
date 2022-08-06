# HPA 基于自定义指标自动扩缩容

offical doc

https://kubernetes.io/zh-cn/docs/tasks/run-application/horizontal-pod-autoscale/



https://glory.blog.csdn.net/article/details/105375822?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.pc_relevant_aa&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.pc_relevant_aa&utm_relevant_index=1



除了基于 CPU 和内存来进行自动扩缩容之外，我们还可以根据自定义的监控指标来进行。这个我们**就需要使用 `Prometheus Adapter`**，Prometheus 用于监控应用的负载和集群本身的各种指标，`Prometheus Adapter` 可以帮我们使用 Prometheus 收集的指标并使用它们来制定扩展策略，这些指标都是通过 APIServer 暴露的，而且 HPA 资源对象也可以很轻易的直接使用。

![img](https://img-blog.csdnimg.cn/20200409174459172.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZseTkxMDkwNQ==,size_16,color_FFFFFF,t_70)custom metrics by prometheus







autoscaling/v1只支持基于CPU指标的缩放；

autoscaling/v2beta1支持Resource Metrics（资源指标，如pod内存）和Custom Metrics（自定义指标）的缩放；

autoscaling/v2beta2支持Resource Metrics（资源指标，如pod的内存）和Custom Metrics（自定义指标）和ExternalMetrics


