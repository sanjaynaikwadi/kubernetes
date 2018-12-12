# Cluster Autoscaler

Cluster Autoscaler (CA) scales your cluster nodes based on pending pods. It periodically checks whether there are any pending pods and increases the size of the cluster if more resources are needed and if the scaled up cluster is still within the user-provided constraints.

In simple word you can say scale up and scale down as demanded.

### High level CA Workflow

![CA WorkFlow](https://github.com/sanjaynaikwadi/kubernetes/blob/master/AutoScaling/CA/CA.png)

