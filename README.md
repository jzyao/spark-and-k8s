# spark on CCE - spark operator

## 准备

## 安装Helm

## 安装Spark operator


```
git clone https://github.com/jzyao/spark-on-k8s-operator.git

cd spark-on-k8s-operator

安装crd
kubectl apply -f manifest/spark-operator-crds.yaml 
\## 安装operator的服务账号与授权策略
kubectl apply -f manifest/spark-operator-rbac.yaml 
\## 安装spark任务的服务账号与授权策略
kubectl apply -f manifest/spark-rbac.yaml 
\## 安装spark-on-k8s-operator 
kubectl apply -f manifest/spark-operator-with-metrics.yaml

```


##

