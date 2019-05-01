# spark-on-k8s-operator实践

Spark Operator是在kubernetes上实践spark的最佳方式，和传统的spark-submit相比提供了更多的故障恢复与可靠性保障，并且提供了监控、日志、UI等能力的集成与支持。

## 准备
- 在华为公有云创建CCE集群，2台工作节点，为了试验方便，每台节点都配了公网IP
  ```
  $ kubectl get node
  NAME            STATUS    ROLES     AGE       VERSION
  192.168.0.164   Ready     <none>    1h        v1.11.7-r0-CCE2.0.20.B001
  192.168.0.171   Ready     <none>    1h        v1.11.7-r0-CCE2.0.20.B001
  ```
- 工作节点安装 `yum install -y socat`
- 下载kubeconfig文件到本地





## 安装Helm
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

## 安装Spark operator
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

## 测试用例
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
查看
![pi](/pic/pi.png?raw=true "pi")

任务结束后，pod`spark-pi-1556724799646-exec-1`销毁，结果传给了drive
```
kubectl logs spark-pi-driver
···
Pi is roughly 3.14039570197851
···
```

##  安装Prometheus
The operator exposes a set of metrics via the metric endpoint to be scraped by Prometheus. The Helm chart by default installs the operator with the additional flag to enable metrics (-enable-metrics=true) as well as other annotations used by Prometheus to scrape the metric endpoint. To install the operator without metrics enabled, pass the appropriate flag during helm install:

创建两块PVC文件存储卷，根据PVC名称修改Prometheus chart/value.yaml中alertmanager和prometheus-server部分
   ![sfs](/pic/sfs.png?raw=true "sfs")

用helm chart安装Prometheus
```
helm install --name cce-prom stable/prometheus
```

可以看到在Prometheus内可以看到如下指标，具体参考https://github.com/jzyao/spark-on-k8s-operator/blob/master/docs/quick-start-guide.md#enable-metric-exporting-to-prometheus

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


## 测试用例
```
jzmac:examples docker$ kubectl apply -f spark-pi-configmap.yaml
sparkapplication.sparkoperator.k8s.io "spark-pi" created
jzmac:examples docker$ kubectl apply -f spark-pi-prometheus.yaml
sparkapplication.sparkoperator.k8s.io "spark-pi" configured
jzmac:examples docker$ kubectl get svc
NAME                                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
cce-prom-prometheus-alertmanager         ClusterIP   10.247.200.68    <none>        80/TCP              20m
cce-prom-prometheus-kube-state-metrics   ClusterIP   None             <none>        80/TCP              20m
cce-prom-prometheus-node-exporter        ClusterIP   None             <none>        9100/TCP            20m
cce-prom-prometheus-pushgateway          ClusterIP   10.247.254.199   <none>        9091/TCP            20m
cce-prom-prometheus-server               ClusterIP   10.247.114.5     <none>        80/TCP              20m
kubernetes                               ClusterIP   10.247.0.1       <none>        443/TCP             3h
prometheus-webui                         NodePort    10.247.61.71     <none>        9090:30344/TCP      13m
spark-pi-1556681717713-driver-svc        ClusterIP   None             <none>        7078/TCP,7079/TCP   5m
spark-pi-ui-svc                          NodePort    10.247.37.29     <none>        4040:31241/TCP      5m
```
![sfs](/pic/sfs.png?raw=true "sfs")


## 安装Spark History Server

