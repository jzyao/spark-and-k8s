# Spark-and-k8s

Not long ago, Kubernetes was added as a natively supported (though still experimental) scheduler for Apache Spark v2.3. This means that you can submit Spark jobs to a Kubernetes cluster using the spark-submit CLI with custom flags, much like the way Spark jobs are submitted to a YARN or Apache Mesos cluster.

Although the Kubernetes support offered by spark-submit is easy to use, there is a lot to be desired in terms of ease of management and monitoring. This is where the Kubernetes Operator for Spark (a.k.a. “the Operator”) comes into play. The Operator tries to provide useful tooling around spark-submit to make running Spark jobs on Kubernetes easier in a production setting, where it matters most. As an implementation of the operator pattern, the Operator extends the Kubernetes API using custom resource definitions (CRDs), which is one of the future directions of Kubernetes.

The purpose of this post is to compare spark-submit and the Operator in terms of functionality, ease of use and user experience. In addition, we would like to provide valuable information to architects, engineers and other interested users of Spark about the options they have when using Spark on Kubernetes along with their pros and cons. Through our journey at Lightbend towards fully supporting fast data pipelines with technologies like Spark on Kubernetes, we would like to communicate what we learned and what is coming next. If you’re short on time, here is a summary of the key points for the busy reader.

What to know about spark-submit:

Part of the Apache Spark project.
First to get updates.
With Spark 3.0, it will close the gap with the Operator regarding arbitrary configuration of Spark pods.
Limited capabilities regarding Spark job management, but some work is still in progress for improving the tool.

What to know about Kubernetes Operator for Spark:

A suite of tools for running Spark jobs on Kubernetes.
The implementation is based on the typical Kubernetes operator pattern.
It uses spark-submit under the hood and hence depends on it.
Supports mounting volumes and ConfigMaps in Spark pods to customize them, a feature that is not available in Apache Spark as of version 2.4.
Provides a useful CLI to manage jobs.

[Spark-Submit](./Spark-Submit)
[Kubernetes Operator for Spark](./spark-on-k8s-operator.md)
