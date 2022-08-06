# Metrics Server



https://glory.blog.csdn.net/article/details/105375822?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.pc_relevant_aa&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.pc_relevant_aa&utm_relevant_index=1

Custom Metrics API

https://github.com/kubernetes/design-proposals-archive/blob/main/instrumentation/custom-metrics-api.md



在 HPA 的第一个版本中，我们需要 `Heapster` 提供 CPU 和[内存](https://so.csdn.net/so/search?q=内存&spm=1001.2101.3001.7020)指标，在 HPA v2 过后就需要安装 Metrcis Server 了，`Metrics Server` 可以通过标准的 Kubernetes [API](https://so.csdn.net/so/search?q=API&spm=1001.2101.3001.7020) 把监控数据暴露出来，有了 `Metrics Server` 之后，我们就完全可以通过标准的 Kubernetes API 来访问我们想要获取的监控数据了：

```bash
https://10.96.0.1/apis/metrics.k8s.io/v1beta1/namespaces/<namespace-name>/pods/<pod-name>
```

比如当我们访问上面的 API 的时候，我们就可以获取到该 Pod 的资源数据，这些数据其实是来自于 kubelet 的 `Summary API` 采集而来的。不过需要说明的是我们这里可以通过标准的 API 来获取资源监控数据，并不是因为 `Metrics Server` 就是 APIServer 的一部分，而是通过 Kubernetes 提供的 `Aggregator` 汇聚插件来实现的，是独立于 APIServer 之外运行的。

how to configure the aggregator later 

https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/



![img](https://img-blog.csdnimg.cn/20200409164957337.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZseTkxMDkwNQ==,size_16,color_FFFFFF,t_70)HPA Metrics Server

 

## 聚合 API

`Aggregator` 允许开发人员编写一个自己的服务，把这个服务注册到 Kubernetes 的 APIServer 里面去，这样我们就可以像原生的 APIServer 提供的 API 使用自己的 API 了，我们把自己的服务运行在 Kubernetes 集群里面，然后 Kubernetes 的 `Aggregator` 通过 Service 名称就可以转发到我们自己写的 Service 里面去了。这样这个聚合层就带来了很多好处：

> - 增加了 API 的扩展性，开发人员可以编写自己的 API 服务来暴露他们想要的 API。
> - 丰富了 API，核心 kubernetes 团队阻止了很多新的 API 提案，通过允许开发人员将他们的 API 作为单独的服务公开，这样就无须社区繁杂的审查了。
> - 开发分阶段实验性 API，新的 API 可以在单独的聚合服务中开发，当它稳定之后，在合并会 APIServer 就很容易了。
> - 确保新 API 遵循 Kubernetes 约定，如果没有这里提出的机制，社区成员可能会被迫推出自己的东西，这样很可能造成社区成员和社区约定不一致。



# Prometheus Adapter for Kubernetes Metrics APIs

[Github repo kubernetes-sigs/prometheus-adapter: An implementation of the custom.metrics.k8s.io API using Prometheus (github.com)](https://github.com/kubernetes-sigs/prometheus-adapter)



This repository contains an implementation of the Kubernetes [resource metrics](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/resource-metrics-api.md), [custom metrics](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/custom-metrics-api.md), and [external metrics](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/external-metrics-api.md) APIs.

This adapter is therefore suitable for use with the autoscaling/v2 Horizontal Pod Autoscaler in Kubernetes 1.6+.
It can also replace the [metrics server](https://github.com/kubernetes-incubator/metrics-server) on clusters that already run Prometheus and collect the appropriate metrics.

# HPA 基于自定义指标自动扩缩容

**除了基于 CPU 和内存来进行自动扩缩容之外，我们还可以根据自定义的监控指标来进行**。这个我们**就需要使用 `Prometheus Adapter`**，Prometheus 用于监控应用的负载和集群本身的各种指标，`Prometheus Adapter` 可以帮我们使用 Prometheus 收集的指标并使用它们来制定扩展策略，这些指标都是通过 APIServer 暴露的，而且 HPA 资源对象也可以很轻易的直接使用。

![img](https://img-blog.csdnimg.cn/20200409174459172.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZseTkxMDkwNQ==,size_16,color_FFFFFF,t_70)custom metrics by prometheus



首先，我们部署一个示例应用，在该应用程序上测试 Prometheus 指标自动缩放，资源清单文件如下所示：（hpa-prome-demo.yaml）

```bash
apiVersion: apps/v1



kind: Deployment



metadata:



  name: hpa-prom-demo



spec:



  selector:



    matchLabels:



      app: nginx-server



  template:



    metadata:



      labels:



        app: nginx-server



    spec:



      containers:



      - name: nginx-demo



        image: cnych/nginx-vts:v1.0



        resources:



          limits:



            cpu: 50m



          requests:



            cpu: 50m



        ports:



        - containerPort: 80



          name: http



---



apiVersion: v1



kind: Service



metadata:



  name: hpa-prom-demo



  annotations:



    prometheus.io/scrape: "true"



    prometheus.io/port: "80"



    prometheus.io/path: "/status/format/prometheus"



spec:



  ports:



  - port: 80



    targetPort: 80



    name: http



  selector:



    app: nginx-server



  type: NodePort
```

这里我们部署的应用是**在 80 端口的 `/status/format/prometheus` 这个端点暴露 nginx-vts 指标的**，前面我们已经在 Prometheus 中配置了 Endpoints 的自动发现，所以我们直接在 Service 对象的 `annotations` 中进行配置，这样我们就可以在 Prometheus 中采集该指标数据了。为了测试方便，我们这里使用 NodePort 类型的 Service，现在直接创建上面的资源对象即可：

```bash
$ kubectl apply -f hpa-prome-demo.yaml



deployment.apps/hpa-prom-demo created



service/hpa-prom-demo created



$ kubectl get pods -l app=nginx-server 



NAME                             READY   STATUS    RESTARTS   AGE



hpa-prom-demo-755bb56f85-lvksr   1/1     Running   0          4m52s



$ kubectl get svc 



NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE



hpa-prom-demo   NodePort    10.101.210.158   <none>        80:32408/TCP   5m44s



......
```

部署完成后我们可以使用如下命令测试应用是否正常，以及指标数据接口能够正常获取：

```bash
$ curl http://k8s.qikqiak.com:32408



<!DOCTYPE html>



<html>



<head>



<title>Welcome to nginx!</title>



<style>



    body {



        width: 35em;



        margin: 0 auto;



        font-family: Tahoma, Verdana, Arial, sans-serif;



    }



</style>



</head>



<body>



<h1>Welcome to nginx!</h1>



<p>If you see this page, the nginx web server is successfully installed and



working. Further configuration is required.</p>



 



<p>For online documentation and support please refer to



<a href="http://nginx.org/">nginx.org</a>.<br/>



Commercial support is available at



<a href="http://nginx.com/">nginx.com</a>.</p>



 



<p><em>Thank you for using nginx.</em></p>



</body>



</html>



$ curl http://k8s.qikqiak.com:32408/status/format/prometheus 



# HELP nginx_vts_info Nginx info



# TYPE nginx_vts_info gauge



nginx_vts_info{hostname="hpa-prom-demo-755bb56f85-lvksr",version="1.13.12"} 1



# HELP nginx_vts_start_time_seconds Nginx start time



# TYPE nginx_vts_start_time_seconds gauge



nginx_vts_start_time_seconds 1586240091.623



# HELP nginx_vts_main_connections Nginx connections



# TYPE nginx_vts_main_connections gauge



......
```

上面的指标数据中，**我们比较关心的是 `nginx_vts_server_requests_total` 这个指标，表示请求总数，是一个 `Counter` 类型的指标，我们将使用该指标的值来确定是否需要对我们的应用进行自动扩缩容**。

 

![img](HPA 详解-基于 CPU、内存和自定义指标自动扩缩容/20200409180447461.png)nginx_vts_server_requests_total

接下来我们**将 Prometheus-Adapter 安装到集群中，并添加一个规则来跟踪 Pod 的请求，我们可以将 Prometheus 中的任何一个指标都用于 HPA**，但是前提是你得通过查询语句将它拿到（包括指标名称和其对应的值）。

这里我们定义一个如下所示的规则：

```bash
rules:



- seriesQuery: 'nginx_vts_server_requests_total'



  seriesFilters: []



  resources:



    overrides:



      kubernetes_namespace:



        resource: namespace



      kubernetes_pod_name:



        resource: pod



  name:



    matches: "^(.*)_total"



    as: "${1}_per_second"



  metricsQuery: (sum(rate(<<.Series>>{<<.LabelMatchers>>}[1m])) by (<<.GroupBy>>))
```

这是一个带参数的 Prometheus 查询，其中：

- `seriesQuery`：查询 Prometheus 的语句，通过这个查询语句查询到的所有指标都可以用于 HPA

- `seriesFilters`：查询到的指标可能会存在不需要的，可以通过它过滤掉。

- `resources`：通过 `seriesQuery` 查询到的只是指标，如果需要查询某个 Pod 的指标，肯定要将它的名称和所在的命名空间作为指标的标签进行查询，`resources` 就是将指标的标签和 k8s 的资源类型关联起来，最常用的就是 pod 和 namespace。有两种添加标签的方式，一种是 `overrides`，另一种是 `template`。

  > - `overrides`：它会将指标中的标签和 k8s 资源关联起来。上面示例中就是将指标中的 pod 和 namespace 标签和 k8s 中的 pod 和 namespace 关联起来，因为 pod 和 namespace 都属于核心 api 组，所以不需要指定 api 组。当我们查询某个 pod 的指标时，它会自动将 pod 的名称和名称空间作为标签加入到查询条件中。比如 `nginx: {group: "apps", resource: "deployment"}` 这么写表示的就是将指标中 nginx 这个标签和 apps 这个 api 组中的 `deployment` 资源关联起来；
  > - template：通过 go 模板的形式。比如`template: "kube_<<.Group>>_<<.Resource>>"` 这么写表示，假如 `<<.Group>>` 为 apps，`<<.Resource>>` 为 deployment，那么它就是将指标中 `kube_apps_deployment` 标签和 deployment 资源关联起来。

- `name`：用来给指标重命名的，之所以要给指标重命名是因为有些指标是只增的，比如以 total 结尾的指标。这些指标拿来做 HPA 是没有意义的，我们一般计算它的速率，以速率作为值，那么此时的名称就不能以 total 结尾了，所以要进行重命名。

  > - `matches`：通过正则表达式来匹配指标名，可以进行分组
  > - `as`：默认值为 `$1`，也就是第一个分组。`as` 为空就是使用默认值的意思。

- `metricsQuery`：这就是 Prometheus 的查询语句了，前面的 `seriesQuery` 查询是获得 HPA 指标。当我们要查某个指标的值时就要通过它指定的查询语句进行了。可以看到查询语句使用了速率和分组，这就是解决上面提到的只增指标的问题。

  > - `Series`：表示指标名称
  > - `LabelMatchers`：附加的标签，目前只有 `pod` 和 `namespace` 两种，因此我们要在之前使用 `resources` 进行关联
  > - `GroupBy`：就是 pod 名称，同样需要使用 `resources` 进行关联。

接下来我们通过 Helm Chart 来部署 Prometheus Adapter，新建 `hpa-prome-adapter-values.yaml` 文件覆盖默认的 Values 值，内容如下所示：

```bash
rules:



  default: false



  custom:



  - seriesQuery: 'nginx_vts_server_requests_total'



    resources: 



      overrides:



        kubernetes_namespace:



          resource: namespace



        kubernetes_pod_name:



          resource: pod



    name:



      matches: "^(.*)_total"



      as: "${1}_per_second"



    metricsQuery: (sum(rate(<<.Series>>{<<.LabelMatchers>>}[1m])) by (<<.GroupBy>>))



 



prometheus:



  url: http://thanos-querier.kube-mon.svc.cluster.local
```

这里我们添加了一条 rules 规则，然后指定了 Prometheus 的地址，我们这里是使用了 Thanos 部署的 Promethues 集群，所以用 Querier 的地址。使用下面的命令一键安装：

```bash
$ helm install prometheus-adapter stable/prometheus-adapter -n kube-mon -f hpa-prome-adapter-values.yaml



NAME: prometheus-adapter



LAST DEPLOYED: Tue Apr  7 15:26:36 2020



NAMESPACE: kube-mon



STATUS: deployed



REVISION: 1



TEST SUITE: None



NOTES:



prometheus-adapter has been deployed.



In a few minutes you should be able to list metrics using the following command(s):



 



  kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1
```

等一小会儿，安装完成后，可以使用下面的命令来检测是否生效了：

```bash
$ kubectl get pods -n kube-mon -l app=prometheus-adapter



NAME                                  READY   STATUS    RESTARTS   AGE



prometheus-adapter-58b559fc7d-l2j6t   1/1     Running   0          3m21s



$  kubectl get --raw="/apis/custom.metrics.k8s.io/v1beta1" | jq



{



  "kind": "APIResourceList",



  "apiVersion": "v1",



  "groupVersion": "custom.metrics.k8s.io/v1beta1",



  "resources": [



    {



      "name": "namespaces/nginx_vts_server_requests_per_second",



      "singularName": "",



      "namespaced": false,



      "kind": "MetricValueList",



      "verbs": [



        "get"



      ]



    },



    {



      "name": "pods/nginx_vts_server_requests_per_second",



      "singularName": "",



      "namespaced": true,



      "kind": "MetricValueList",



      "verbs": [



        "get"



      ]



    }



  ]



}
```

我们可以看到 `nginx_vts_server_requests_per_second` 指标可用。 现在，让我们检查该指标的当前值：

```bash
$ kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/nginx_vts_server_requests_per_second" | jq .



{



  "kind": "MetricValueList",



  "apiVersion": "custom.metrics.k8s.io/v1beta1",



  "metadata": {



    "selfLink": "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/%2A/nginx_vts_server_requests_per_second"



  },



  "items": [



    {



      "describedObject": {



        "kind": "Pod",



        "namespace": "default",



        "name": "hpa-prom-demo-755bb56f85-lvksr",



        "apiVersion": "/v1"



      },



      "metricName": "nginx_vts_server_requests_per_second",



      "timestamp": "2020-04-07T09:45:45Z",



      "value": "527m",



      "selector": null



    }



  ]



}
```

出现类似上面的信息就表明已经配置成功了，接下来我们部署一个针对上面的自定义指标的 HPA资源对象，如下所示：(hpa-prome.yaml)

```bash
apiVersion: autoscaling/v2beta1



kind: HorizontalPodAutoscaler



metadata:



  name: nginx-custom-hpa



spec:



  scaleTargetRef:



    apiVersion: apps/v1



    kind: Deployment



    name: hpa-prom-demo



  minReplicas: 2



  maxReplicas: 5



  metrics:



  - type: Pods



    pods:



      metricName: nginx_vts_server_requests_per_second



      targetAverageValue: 10
```

如果请求数超过每秒10个，则将对应用进行扩容。直接创建上面的资源对象：

```bash
$ kubectl apply -f hpa-prome.yaml



horizontalpodautoscaler.autoscaling/nginx-custom-hpa created



$ kubectl describe hpa nginx-custom-hpa



Name:                                              nginx-custom-hpa



Namespace:                                         default



Labels:                                            <none>



Annotations:                                       kubectl.kubernetes.io/last-applied-configuration:



                                                     {"apiVersion":"autoscaling/v2beta1","kind":"HorizontalPodAutoscaler","metadata":{"annotations":{},"name":"nginx-custom-hpa","namespace":"d...



CreationTimestamp:                                 Tue, 07 Apr 2020 17:54:55 +0800



Reference:                                         Deployment/hpa-prom-demo



Metrics:                                           ( current / target )



  "nginx_vts_server_requests_per_second" on pods:  <unknown> / 10



Min replicas:                                      2



Max replicas:                                      5



Deployment pods:                                   1 current / 2 desired



Conditions:



  Type         Status  Reason            Message



  ----         ------  ------            -------



  AbleToScale  True    SucceededRescale  the HPA controller was able to update the target scale to 2



Events:



  Type    Reason             Age   From                       Message



  ----    ------             ----  ----                       -------



  Normal  SuccessfulRescale  7s    horizontal-pod-autoscaler  New size: 2; reason: Current number of replicas below Spec.MinReplicas
```

可以看到 HPA 对象已经生效了，会应用最小的副本数2，所以会新增一个 Pod 副本：

```bash
$ kubectl get pods -l app=nginx-server



NAME                             READY   STATUS    RESTARTS   AGE



hpa-prom-demo-755bb56f85-s5dzf   1/1     Running   0          67s



hpa-prom-demo-755bb56f85-wbpfr   1/1     Running   0          3m30s
```

接下来我们同样对应用进行压测：

```bash
$ while true; do wget -q -O- http://k8s.qikqiak.com:32408; done
```

打开另外一个终端观察 HPA 对象的变化：

```bash
$ kubectl get hpa



NAME               REFERENCE                  TARGETS     MINPODS   MAXPODS   REPLICAS   AGE



nginx-custom-hpa   Deployment/hpa-prom-demo   14239m/10   2         5         2          4m27s



$ kubectl describe hpa nginx-custom-hpa



Name:                                              nginx-custom-hpa



Namespace:                                         default



Labels:                                            <none>



Annotations:                                       kubectl.kubernetes.io/last-applied-configuration:



                                                     {"apiVersion":"autoscaling/v2beta1","kind":"HorizontalPodAutoscaler","metadata":{"annotations":{},"name":"nginx-custom-hpa","namespace":"d...



CreationTimestamp:                                 Tue, 07 Apr 2020 17:54:55 +0800



Reference:                                         Deployment/hpa-prom-demo



Metrics:                                           ( current / target )



  "nginx_vts_server_requests_per_second" on pods:  14308m / 10



Min replicas:                                      2



Max replicas:                                      5



Deployment pods:                                   3 current / 3 desired



Conditions:



  Type            Status  Reason              Message



  ----            ------  ------              -------



  AbleToScale     True    ReadyForNewScale    recommended size matches current size



  ScalingActive   True    ValidMetricFound    the HPA was able to successfully calculate a replica count from pods metric nginx_vts_server_requests_per_second



  ScalingLimited  False   DesiredWithinRange  the desired count is within the acceptable range



Events:



  Type    Reason             Age   From                       Message



  ----    ------             ----  ----                       -------



  Normal  SuccessfulRescale  5m2s  horizontal-pod-autoscaler  New size: 2; reason: Current number of replicas below Spec.MinReplicas



  Normal  SuccessfulRescale  61s   horizontal-pod-autoscaler  New size: 3; reason: pods metric nginx_vts_server_requests_per_second above target
```

可以看到指标 `nginx_vts_server_requests_per_second` 的数据已经超过阈值了，触发扩容动作了，副本数变成了3，但是后续很难继续扩容了，这是因为上面我们的 `while` 命令并不够快，3个副本完全可以满足每秒不超过 10 个请求的阈值。

![img](HPA 详解-基于 CPU、内存和自定义指标自动扩缩容/20200409181457991-1658893152235.png)nginx_vts_server_requests_per_second

如果需要更好的进行测试，我们可以使用一些压测工具，比如 ab、fortio 等工具。当我们中断测试后，默认5分钟过后就会自动缩容：

```bash
$ kubectl describe hpa nginx-custom-hpa



Name:                                              nginx-custom-hpa



Namespace:                                         default



Labels:                                            <none>



Annotations:                                       kubectl.kubernetes.io/last-applied-configuration:



                                                     {"apiVersion":"autoscaling/v2beta1","kind":"HorizontalPodAutoscaler","metadata":{"annotations":{},"name":"nginx-custom-hpa","namespace":"d...



CreationTimestamp:                                 Tue, 07 Apr 2020 17:54:55 +0800



Reference:                                         Deployment/hpa-prom-demo



Metrics:                                           ( current / target )



  "nginx_vts_server_requests_per_second" on pods:  533m / 10



Min replicas:                                      2



Max replicas:                                      5



Deployment pods:                                   2 current / 2 desired



Conditions:



  Type            Status  Reason            Message



  ----            ------  ------            -------



  AbleToScale     True    ReadyForNewScale  recommended size matches current size



  ScalingActive   True    ValidMetricFound  the HPA was able to successfully calculate a replica count from pods metric nginx_vts_server_requests_per_second



  ScalingLimited  True    TooFewReplicas    the desired replica count is less than the minimum replica count



Events:



  Type    Reason             Age   From                       Message



  ----    ------             ----  ----                       -------



  Normal  SuccessfulRescale  23m   horizontal-pod-autoscaler  New size: 2; reason: Current number of replicas below Spec.MinReplicas



  Normal  SuccessfulRescale  19m   horizontal-pod-autoscaler  New size: 3; reason: pods metric nginx_vts_server_requests_per_second above target



  Normal  SuccessfulRescale  4m2s  horizontal-pod-autoscaler  New size: 2; reason: All metrics below target
```

到这里我们就完成了使用自定义的指标对应用进行自动扩缩容的操作。如果 Prometheus 安装在我们的 Kubernetes 集群之外，则只需要确保可以从集群访问到查询的端点，并在 adapter 的部署清单中对其进行更新即可。在更复杂的场景中，可以获取多个指标结合使用来制定扩展策略。