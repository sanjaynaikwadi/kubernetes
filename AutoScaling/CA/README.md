# Cluster Autoscaler

Cluster Autoscaler (CA) scales your cluster nodes based on pending pods. It periodically checks whether there are any pending pods and increases the size of the cluster if more resources are needed and if the scaled up cluster is still within the user-provided constraints.

In simple word you can say scale up and scale down of nodes as demanded.

### High level CA Workflow

![CA WorkFlow](https://github.com/sanjaynaikwadi/kubernetes/blob/master/AutoScaling/CA/CA.png)

### Prerequisites

- Kubernetes Cluster (I tested with 1.10.11 via KOPS on AWS)
- Resource limit set in deployment with HPA enabled 
- Metric Server installed

[How to Setup Metric Server](https://github.com/sanjaynaikwadi/kubernetes/tree/master/AutoScaling/Metric-Server)

[How to Setup HPA](https://github.com/sanjaynaikwadi/kubernetes/tree/master/AutoScaling/HPA)

Assuming you have the below setup in working mode to carry out the testing, I have used AWS enviornment and used t2.micro instances for testing. I have also configured a seperate instance group (db-pool) for testing

```
Validating cluster houm-host-devops-11-sg.k8s.local

INSTANCE GROUPS
NAME                    ROLE    MACHINETYPE     MIN     MAX     SUBNETS
bastions                Bastion t2.micro        1       1       utility-ap-southeast-1a,utility-ap-southeast-1b,utility-ap-southeast-1c
db-pool                 Node    t2.micro        1       4       ap-southeast-1a
master-ap-southeast-1a  Master  m4.large        1       1       ap-southeast-1a
nodes                   Node    t2.micro        1       1       ap-southeast-1a,ap-southeast-1b,ap-southeast-1c

NODE STATUS
NAME                                                    ROLE    READY
ip-172-20-116-132.ap-southeast-1.compute.internal       node    True
ip-172-20-52-218.ap-southeast-1.compute.internal        node    True
ip-172-20-59-245.ap-southeast-1.compute.internal        master  True

Your cluster houm-host-devops-11-sg.k8s.local is ready
```

### Lets install the auto-scaler

- Clone or download the yaml file and make necessary changes in file, you can specify multiple instance group ASG

```
            - --nodes=1:2:db-pool.houm-host-devops-11-sg.k8s.local
```
Let me explain what all fields are :
- nodes - common name
- 1:2 - Minimum : Maxmium node used for scale up and scale down
- db-pool - ASG name
- houm-host-devops-11-sg.k8s.local - Cluster name 

You can also use aws cli to list the autoscaling group 
```
aws autoscaling describe-auto-scaling-groups |grep AutoScalingGroupName
```

NOTE : I have been using cluster created with KOPS it attaches the auto-scaling policy to all ASG associated with cluster, or else you can use the policy uploaded and attach to your ASG in AWS

### Lets deploy the HPA
[HPA]()

Use the HPA which is explained and you will see at one point when your pods goes in pending state your new node will be spawned.

- Verify with 
```
"kubectl get pods"
"kubectl get nodes"
```

Once you stop the load, your newly spawn node will be deleted after the cooldown period.

### Logs to Verify in Cluster Pod

- New node Spwan

- One node deleted after the load is normal
```
[root@localhost ~]# kubectl get no
NAME                                                STATUS    ROLES     AGE       VERSION
ip-172-20-116-132.ap-southeast-1.compute.internal   Ready     node      12d       v1.10.11
**ip-172-20-50-247.ap-southeast-1.compute.internal    Ready     node      1d        v1.10.11**
ip-172-20-52-218.ap-southeast-1.compute.internal    Ready     node      7d        v1.10.11
ip-172-20-59-245.ap-southeast-1.compute.internal    Ready     master    12d       v1.10.11
```

```
I1217 16:50:18.084534       1 static_autoscaler.go:355] Starting scale down
**I1217 16:50:18.197431       1 scale_down.go:387] ip-172-20-50-247.ap-southeast-1.compute.internal was unneeded for 3m53.410923843s**
I1217 16:50:18.197520       1 scale_down.go:446] No candidates for scale down
I1217 16:50:18.666667       1 leaderelection.go:199] successfully renewed lease kube-system/cluster-autoscaler
I1217 16:50:20.763498       1 leaderelection.go:199] successfully renewed lease kube-system/cluster-autoscaler
I1217 16:50:22.866695       1 leaderelection.go:199] successfully renewed lease kube-system/cluster-autoscaler
I1217 16:50:23.073018       1 reflector.go:428] k8s.io/autoscaler/cluster-autoscaler/vendor/k8s.io/client-go/informers/factory.go:87: Watch close - *v1beta1.ReplicaSet total 1 items received
I1217 16:50:24.872079       1 leaderelection.go:199] successfully renewed lease kube-system/cluster-autoscaler
I1217 16:50:26.971706       1 leaderelection.go:199] successfully renewed lease kube-system/cluster-autoscaler
I1217 16:50:28.204189       1 static_autoscaler.go:114] Starting main loop
I1217 16:50:28.204280       1 aws_manager.go:241] Refreshed ASG list, next refresh after 2018-12-17 16:51:28.204274635 +0000 UTC
```

Successfully removed after 10 mins wait period
```
I1217 16:56:29.589379       1 static_autoscaler.go:355] Starting scale down
I1217 16:56:29.712467       1 scale_down.go:387] ip-172-20-50-247.ap-southeast-1.compute.internal was unneeded for 10m4.865295051s
I1217 16:56:29.718530       1 reflector.go:428] k8s.io/autoscaler/cluster-autoscaler/utils/kubernetes/listers.go:293: Watch close - *v1beta1.DaemonSet total 0 items received
I1217 16:56:29.799243       1 scale_down.go:594] Scale-down: removing empty node ip-172-20-50-247.ap-southeast-1.compute.internal
I1217 16:56:29.810031       1 factory.go:33] Event(v1.ObjectReference{Kind:"ConfigMap", Namespace:"kube-system", Name:"cluster-autoscaler-status", UID:"0babf809-fc84-11e8-a5f0-02375222ee44", APIVersion:"v1", ResourceVersion:"1790247", FieldPath:""}): type: 'Normal' reason: 'ScaleDownEmpty' Scale-down: removing empty node ip-172-20-50-247.ap-southeast-1.compute.internal
I1217 16:56:29.882824       1 delete.go:53] Successfully added toBeDeletedTaint on node ip-172-20-50-247.ap-southeast-1.compute.internal
I1217 16:56:30.269519       1 aws_manager.go:341] Terminating EC2 instance: i-06a95d8b15b9ccbc7
I1217 16:56:30.270735       1 factory.go:33] Event(v1.ObjectReference{Kind:"Node", Namespace:"", Name:"ip-172-20-50-247.ap-southeast-1.compute.internal", UID:"a2db0b9e-012c-11e9-a5f0-02375222ee44", APIVersion:"v1", ResourceVersion:"1790249", FieldPath:""}): type: 'Normal' reason: 'ScaleDown' node removed by cluster autoscaler
```

```
[root@localhost ~]# kubectl get no
NAME                                                STATUS    ROLES     AGE       VERSION
ip-172-20-116-132.ap-southeast-1.compute.internal   Ready     node      12d       v1.10.11
ip-172-20-52-218.ap-southeast-1.compute.internal    Ready     node      7d        v1.10.11
ip-172-20-59-245.ap-southeast-1.compute.internal    Ready     master    12d       v1.10.11
```

### Reference links
https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler/cloudprovider/aws

https://kumorilabs.com/blog/k8s-5-setup-horizontal-pod-cluster-autoscaling-kubernetes/

https://medium.com/magalix/kubernetes-autoscaling-101-cluster-autoscaler-horizontal-pod-autoscaler-and-vertical-pod-2a441d9ad231

