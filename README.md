# spark on CCE - spark operator

## 准备
- 创建CCE集群，2台工作节点，为了试验方便，每台节点都配了公网IP
  ```
  # kubectl get node
  NAME            STATUS    ROLES     AGE       VERSION
  192.168.0.135   Ready     <none>    14m       v1.11.7-r0-CCE2.0.20.B001
  192.168.0.34    Ready     <none>    14m       v1.11.7-r0-CCE2.0.20.B001
  ```
- 创建两块文件存储卷，安装Prometheus时需要
   ![sfs](/pic/sfs.png?raw=true "sfs")
- 工作节点安装 `yum install -y socat`
- 下载kubeconfig文件到本地





## 安装Helm
```
wget -O helm.tar.gz https://storage.googleapis.com/kubernetes-helm/helm-v2.13.0-linux-amd64.tar.gz
tar -zxvf helm.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
helm version
helm repo add stable https://kubernetes.oss-cn-hangzhou.aliyuncs.com/
helm repo add incubator http://storage.googleapis.com/kubernetes-charts-incubator

kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'

helm init --upgrade --service-account tiller --tiller-image=sapcc/tiller:v2.13.0
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

##  安装Prometheus
用helm chart安装Pormetheus
```
helm install --name cce-prom stable/prometheus```
在这一步，有两个容器会报错，报错为，原因是
解决方法
修改cce-prom-prometheus-alertmanager 和cce-prom-prometheus-server 两个deployment的yaml文件中persistentVolumeClaim名称，用之前准备的两块文件存储卷名替换


kubectl expose deployment cce-prom-prometheus-server --type=NodePort  --name=prometheus-webui
```
## 测试用例

## 安装Spark History Server

