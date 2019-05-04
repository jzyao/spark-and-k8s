# spark-on-k8s-operator
## Table of Contents
* [Introduction](#Introduction)
* [Architecture](#architecture)
* [部署测试](#部署测试)

[官方文档](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator)

Spark Operator是在kubernetes上实践spark的最佳方式，和传统的spark-submit相比提供了更多的故障恢复与可靠性保障，并且提供了监控、日志、UI等能力的集成与支持。

## Introduction

In Spark 2.3, Kubernetes becomes an official scheduler backend for Spark, additionally to the standalone scheduler, Mesos, and Yarn. Compared with the alternative approach of deploying a standalone Spark cluster on top of Kubernetes and submit applications to run on the standalone cluster, having Kubernetes as a native scheduler backend offers some important benefits as discussed in [SPARK-18278](https://issues.apache.org/jira/browse/SPARK-18278) and is a huge leap forward. However, the way life cycle of Spark applications are managed, e.g., how applications get submitted to run on Kubernetes and how application status is tracked, are vastly different from that of other types of workloads on Kubernetes, e.g., Deployments, DaemonSets, and StatefulSets. The Kubernetes Operator for Apache Spark reduces the gap and allow Spark applications to be specified, run, and monitored idiomatically on Kubernetes.

Specifically, the Kubernetes Operator for Apache Spark follows the recent trend of leveraging the [operator](https://coreos.com/blog/introducing-operators.html) pattern for managing the life cycle of Spark applications on a Kubernetes cluster. The operator allows Spark applications to be specified in a declarative manner (e.g., in a YAML file) and run without the need to deal with the spark submission process. It also enables status of Spark applications to be tracked and presented idiomatically like other types of workloads on Kubernetes. This document discusses the design and architecture of the operator. For documentation of the [CustomResourceDefinition](https://kubernetes.io/docs/concepts/api-extension/custom-resources/) for specification of Spark applications, please refer to [API Definition](api.md)      

## Architecture

The operator consists of:
* a `SparkApplication` controller that watches events of creation, updates, and deletion of 
`SparkApplication` objects and acts on the watch events,
* a *submission runner* that runs `spark-submit` for submissions received from the controller,
* a *Spark pod monitor* that watches for Spark pods and sends pod status updates to the controller,
* a [Mutating Admission Webhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/) that handles customizations for Spark driver and executor pods based on the annotations on the pods added by the controller,
* and also a command-line tool named `sparkctl` for working with the operator. 

The following diagram shows how different components interact and work together.

![Architecture Diagram](./pic/architecture-diagram.png)

Specifically, a user uses the `sparkctl` (or `kubectl`) to create a `SparkApplication` object. The `SparkApplication` controller receives the object through a watcher from the API server, creates a submission carrying the `spark-submit` arguments, and sends the submission to the *submission runner*. The submission runner submits the application to run and creates the driver pod of the application. Upon starting, the driver pod creates the executor pods. While the application is running, the *Spark pod monitor* watches the pods of the application and sends status updates of the pods back to the controller, which then updates the status of the application accordingly. 


## 部署测试
### 准备
- 在华为公有云创建CCE集群，2台工作节点，为了试验方便，每台节点都配了公网IP
  ```
  $ kubectl get node
  NAME            STATUS    ROLES     AGE       VERSION
  192.168.0.164   Ready     <none>    1h        v1.11.7-r0-CCE2.0.20.B001
  192.168.0.171   Ready     <none>    1h        v1.11.7-r0-CCE2.0.20.B001
  ```
- 工作节点安装 `yum install -y socat`
- 下载kubeconfig文件到本地

### 安装Helm
```
wget -O helm.tar.gz https://storage.googleapis.com/kubernetes-helm/helm-v2.13.0-linux-amd64.tar.gz
tar -zxvf helm.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
helm version
# helm repo add stable https://kubernetes.oss-cn-hangzhou.aliyuncs.com/
# helm repo add incubator http://storage.googleapis.com/kubernetes-charts-incubator
helm init --upgrade --service-account tiller --tiller-image=sapcc/tiller:v2.13.0
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'

```

### 安装Spark operator
官方的文档是通过Helm Chart进行安装的，由于很多开发者的环境无法连通google的repo，因此此处我们通过标准的yaml进行安装。
```
git clone https://github.com/jzyao/spark-on-k8s-operator.git
cd spark-on-k8s-operator

##安装crd
kubectl apply -f manifest/spark-operator-crds.yaml 
## 安装operator的服务账号与授权策略
kubectl apply -f manifest/spark-operator-rbac.yaml 
## 安装spark任务的服务账号与授权策略
kubectl apply -f manifest/spark-rbac.yaml 
## 安装spark-on-k8s-operator 
kubectl apply -f manifest/spark-operator-with-metrics.yaml
```

```
kubectl get all -n spark-operator
NAME                            READY     STATUS    RESTARTS   AGE
sparkoperator-d5c8869bd-4n8q4   1/1       Running   0          39m

NAME            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
sparkoperator   1         1         1            1           39m

NAME                      DESIRED   CURRENT   READY     AGE
sparkoperator-d5c8869bd   1         1         1         39m
```

### 测试用例
跑一个Spark的helloworld例子，圆周率运行
```
jzmac:examples docker$ kubectl apply -f spark-pi.yaml
sparkapplication.sparkoperator.k8s.io "spark-pi" created
jzmac:examples docker$ kubectl get sparkapplication
NAME       AGE
spark-pi   21s
jzmac:examples docker$ kubectl get all
NAME                            READY     STATUS    RESTARTS   AGE
spark-pi-1556724799646-exec-1   1/1       Running   0          12s
spark-pi-driver                 1/1       Running   0          3m

NAME                                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
kubernetes                          ClusterIP   10.247.0.1      <none>        443/TCP             22m
spark-pi-1556724799646-driver-svc   ClusterIP   None            <none>        7078/TCP,7079/TCP   3m
spark-pi-ui-svc                     NodePort    10.247.123.31   <none>        4040:31410/TCP      3m
```
查看结果

![pi](./pic/pi.png?raw=true "pi")

任务结束后，pod`spark-pi-1556724799646-exec-1`销毁，结果传给了driver
```
kubectl logs spark-pi-driver
···
Pi is roughly 3.14039570197851
···
```

###  安装Prometheus
The operator exposes a set of metrics via the metric endpoint to be scraped by Prometheus. The Helm chart by default installs the operator with the additional flag to enable metrics (-enable-metrics=true) as well as other annotations used by Prometheus to scrape the metric endpoint. To install the operator without metrics enabled, pass the appropriate flag during helm install:

创建两块PVC文件存储卷，根据PVC名称修改Prometheus chart/value.yaml中alertmanager和prometheus-server部分

   ![sfs](./pic/sfs.png?raw=true "sfs")

用helm chart安装Prometheus
```
helm install --name cce-prom stable/prometheus
```

Prometheus内target已UP

![metrics](./pic/metrics.png?raw=true "metrics")

Spark相关数据已经可以采集到

![target](./pic/target.png?raw=true "target")

指标列表如下，具体参考[官方文档](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/quick-start-guide.md)

| Metric | Description |
| ------------- | ------------- |
| `spark_app_submit_count`  | Total number of SparkApplication submitted by the Operator.|
| `spark_app_success_count` | Total number of SparkApplication which completed successfully.|
| `spark_app_failure_count` | Total number of SparkApplication which failed to complete. |
| `spark_app_running_count` | Total number of SparkApplication which are currently running.|
| `spark_app_success_execution_time_microseconds` | Execution time for applications which succeeded.|
| `spark_app_failure_execution_time_microseconds` |Execution time for applications which failed. |
| `spark_app_executor_success_count` | Total number of Spark Executors which completed successfully. |
| `spark_app_executor_failure_count` | Total number of Spark Executors which failed. |
| `spark_app_executor_running_count` | Total number of Spark Executors which are currently running. |

### 监控，日志管理
配合华为云AOM可以实时监控，和查看实时/历史日志

dashboard部分

![dashboard](./pic/dashboard.png?raw=true "dashboard")

日志部分

![log](./pic/log.png?raw=true "log")

### 安装Spark History Server

