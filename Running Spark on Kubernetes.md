# Running Spark on Kubernetes

[官方文档](https://github.com/apache-spark-on-k8s/spark)
[apache-spark-on-k8s](https://github.com/apache-spark-on-k8s/spark)


## Prerequisites
A runnable distribution of Spark 2.3 or above.
A running Kubernetes cluster at version >= 1.6 with access configured to it using kubectl. If you do not already have a working Kubernetes cluster, you may set up a test cluster on your local machine using minikube.
We recommend using the latest release of minikube with the DNS addon enabled.
Be aware that the default minikube configuration is not enough for running Spark applications. We recommend 3 CPUs and 4g of memory to be able to start a simple Spark application with a single executor.
You must have appropriate permissions to list, create, edit and delete pods in your cluster. You can verify that you can list these resources by running kubectl auth can-i <list|create|edit|delete> pods.
The service account credentials used by the driver pods must be allowed to create pods, services and configmaps.
You must have Kubernetes DNS configured in your cluster.

## How it works
 ![k8s-cluster-mode](./pic/k8s-cluster-mode.png?raw=true "k8s-cluster-mode.png")
 spark-submit can be directly used to submit a Spark application to a Kubernetes cluster. The submission mechanism works as follows:

Spark creates a Spark driver running within a Kubernetes pod.
The driver creates executors which are also running within Kubernetes pods and connects to them, and executes application code.
When the application completes, the executor pods terminate and are cleaned up, but the driver pod persists logs and remains in “completed” state in the Kubernetes API until it’s eventually garbage collected or manually cleaned up.
Note that in the completed state, the driver pod does not use any computational or memory resources.

The driver and executor pod scheduling is handled by Kubernetes. It is possible to schedule the driver and executor pods on a subset of available nodes through a node selector using the configuration property for it. It will be possible to use more advanced scheduling hints like node/pod affinities in a future release.

## Submitting Applications to Kubernetes


### Docker Images
Kubernetes requires users to supply images that can be deployed into containers within pods. The images are built to be run in a container runtime environment that Kubernetes supports. Docker is a container runtime environment that is frequently used with Kubernetes. Spark (starting with version 2.3) ships with a Dockerfile that can be used for this purpose, or customized to match an individual application’s needs. It can be found in the kubernetes/dockerfiles/ directory.

Spark also ships with a bin/docker-image-tool.sh script that can be used to build and publish the Docker images to use with the Kubernetes backend.

Example usage is:
```
$ ./bin/docker-image-tool.sh -r <repo> -t my-tag build
$ ./bin/docker-image-tool.sh -r <repo> -t my-tag push
```
### Cluster Mode
To launch Spark Pi in cluster mode,
首先设置RBAC
```
kubectl create serviceaccount spark
kubectl create clusterrolebinding spark-role --clusterrole=edit  --serviceaccount=default:spark --namespace=default
```
提交作业
```
./spark-submit \
    --master k8s://https://119.3.100.33:5443 \
     --deploy-mode cluster  \
    --conf spark.executor.instances=1  \
    --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark  \
    --conf spark.kubernetes.container.image=jzdoris/spark:v2.4.0  \
    --class org.apache.spark.examples.SparkPi  \
    --name spark-pi  \
    local:///opt/spark/examples/jars/spark-examples_2.11-2.4.0.jar
```

实际结果
```
$ ./spark-submit \
>     --master k8s://https://119.3.100.33:5443 \
>      --deploy-mode cluster  \
>     --conf spark.executor.instances=1  \
>     --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark  \
>     --conf spark.kubernetes.container.image=jzdoris/spark:v2.4.0  \
>     --class org.apache.spark.examples.SparkPi  \
>     --name spark-pi  \
>     local:///opt/spark/examples/jars/spark-examples_2.11-2.4.0.jar
log4j:WARN No appenders could be found for logger (io.fabric8.kubernetes.client.Config).
log4j:WARN Please initialize the log4j system properly.
log4j:WARN See http://logging.apache.org/log4j/1.2/faq.html#noconfig for more info.
Using Spark's default log4j profile: org/apache/spark/log4j-defaults.properties
19/05/03 14:51:30 INFO LoggingPodStatusWatcherImpl: State changed, new state:
	 pod name: spark-pi-1556866289210-driver
	 namespace: default
	 labels: spark-app-selector -> spark-4421d41567514da1a3116124ef47c17f, spark-role -> driver
	 pod uid: e1773f91-6d6f-11e9-ba6b-fa163e8dc0da
	 creation time: 2019-05-03T06:51:30Z
	 service account name: spark
	 volumes: spark-local-dir-1, spark-conf-volume, spark-token-nj8pq
	 node name: N/A
	 start time: N/A
	 container images: N/A
	 phase: Pending
	 status: []
19/05/03 14:51:30 INFO LoggingPodStatusWatcherImpl: State changed, new state:
	 pod name: spark-pi-1556866289210-driver
	 namespace: default
	 labels: spark-app-selector -> spark-4421d41567514da1a3116124ef47c17f, spark-role -> driver
	 pod uid: e1773f91-6d6f-11e9-ba6b-fa163e8dc0da
	 creation time: 2019-05-03T06:51:30Z
	 service account name: spark
	 volumes: spark-local-dir-1, spark-conf-volume, spark-token-nj8pq
	 node name: 192.168.0.164
	 start time: N/A
	 container images: N/A
	 phase: Pending
	 status: []
19/05/03 14:51:30 INFO LoggingPodStatusWatcherImpl: State changed, new state:
	 pod name: spark-pi-1556866289210-driver
	 namespace: default
	 labels: spark-app-selector -> spark-4421d41567514da1a3116124ef47c17f, spark-role -> driver
	 pod uid: e1773f91-6d6f-11e9-ba6b-fa163e8dc0da
	 creation time: 2019-05-03T06:51:30Z
	 service account name: spark
	 volumes: spark-local-dir-1, spark-conf-volume, spark-token-nj8pq
	 node name: 192.168.0.164
	 start time: 2019-05-03T06:51:30Z
	 container images: jzdoris/spark:v2.4.0
	 phase: Pending
	 status: [ContainerStatus(containerID=null, image=jzdoris/spark:v2.4.0, imageID=, lastState=ContainerState(running=null, terminated=null, waiting=null, additionalProperties={}), name=spark-kubernetes-driver, ready=false, restartCount=0, state=ContainerState(running=null, terminated=null, waiting=ContainerStateWaiting(message=null, reason=ContainerCreating, additionalProperties={}), additionalProperties={}), additionalProperties={})]
19/05/03 14:51:30 INFO Client: Waiting for application spark-pi to finish...
19/05/03 14:51:33 INFO LoggingPodStatusWatcherImpl: State changed, new state:
	 pod name: spark-pi-1556866289210-driver
	 namespace: default
	 labels: spark-app-selector -> spark-4421d41567514da1a3116124ef47c17f, spark-role -> driver
	 pod uid: e1773f91-6d6f-11e9-ba6b-fa163e8dc0da
	 creation time: 2019-05-03T06:51:30Z
	 service account name: spark
	 volumes: spark-local-dir-1, spark-conf-volume, spark-token-nj8pq
	 node name: 192.168.0.164
	 start time: 2019-05-03T06:51:30Z
	 container images: jzdoris/spark:v2.4.0
	 phase: Running
	 status: [ContainerStatus(containerID=docker://25424626564f6029d9424d3dc2cf090e4a1b0223429b6714543121035e2b8d72, image=jzdoris/spark:v2.4.0, imageID=docker-pullable://jzdoris/spark@sha256:375cd079d14179687b37625ce850c905c452010325d62c35618b8e0c8ddc749b, lastState=ContainerState(running=null, terminated=null, waiting=null, additionalProperties={}), name=spark-kubernetes-driver, ready=true, restartCount=0, state=ContainerState(running=ContainerStateRunning(startedAt=2019-05-03T06:51:32Z, additionalProperties={}), terminated=null, waiting=null, additionalProperties={}), additionalProperties={})]
19/05/03 14:51:49 INFO LoggingPodStatusWatcherImpl: State changed, new state:
	 pod name: spark-pi-1556866289210-driver
	 namespace: default
	 labels: spark-app-selector -> spark-4421d41567514da1a3116124ef47c17f, spark-role -> driver
	 pod uid: e1773f91-6d6f-11e9-ba6b-fa163e8dc0da
	 creation time: 2019-05-03T06:51:30Z
	 service account name: spark
	 volumes: spark-local-dir-1, spark-conf-volume, spark-token-nj8pq
	 node name: 192.168.0.164
	 start time: 2019-05-03T06:51:30Z
	 container images: jzdoris/spark:v2.4.0
	 phase: Succeeded
	 status: [ContainerStatus(containerID=docker://25424626564f6029d9424d3dc2cf090e4a1b0223429b6714543121035e2b8d72, image=jzdoris/spark:v2.4.0, imageID=docker-pullable://jzdoris/spark@sha256:375cd079d14179687b37625ce850c905c452010325d62c35618b8e0c8ddc749b, lastState=ContainerState(running=null, terminated=null, waiting=null, additionalProperties={}), name=spark-kubernetes-driver, ready=false, restartCount=0, state=ContainerState(running=null, terminated=ContainerStateTerminated(containerID=docker://25424626564f6029d9424d3dc2cf090e4a1b0223429b6714543121035e2b8d72, exitCode=0, finishedAt=2019-05-03T06:51:48Z, message=null, reason=Completed, signal=null, startedAt=2019-05-03T06:51:32Z, additionalProperties={}), waiting=null, additionalProperties={}), additionalProperties={})]
19/05/03 14:51:49 INFO LoggingPodStatusWatcherImpl: Container final statuses:


	 Container name: spark-kubernetes-driver
	 Container image: jzdoris/spark:v2.4.0
	 Container state: Terminated
	 Exit code: 0
19/05/03 14:51:49 INFO Client: Application spark-pi finished.
19/05/03 14:51:49 INFO ShutdownHookManager: Shutdown hook called
19/05/03 14:51:49 INFO ShutdownHookManager: Deleting directory /private/var/folders/yw/c18tkkms4qv_f8hf_6tp9dcc0000gn/T/spark-33549fe4-bbac-4db5-ab32-fa46bed8d4c7
```
检查日志结果
查看pod日志，得到Pi结果
 ![rizhi](./pic/rizhi.jpg?raw=true "rizhi")
