# spark on CCE - spark operator

## 准备
- 创建CCE集群
- 创建两块文件存储卷，安装Prometheus时需要
- 工作节点安装 `yum install -y socat`
- 下载kubeconfig文件到本地


## 安装Helm
```
wget -O helm.tar.gz https://storage.googleapis.com/kubernetes-helm/helm-v2.11.0-linux-amd64.tar.gz
tar -zxvf helm-v2.11.0-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
helm version
helm init --tiller-image=sapcc/tiller:v2.11.0 --service-account tiller

```

## 安装Spark operator


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


##  安装Prometheus
用helm chart安装Pormetheus
```
helm install --name CCE-prom stable/prometheus
```
在这一步，有两个容器会报错，报错为，原因是
解决方法
修改deployment的yaml文件，用之前准备的两块文件存储卷名替换

```
kubectl expose deployment --type=NodePort  --name=prometheus-webui
```
## 测试用例

## 安装Spark History Server

