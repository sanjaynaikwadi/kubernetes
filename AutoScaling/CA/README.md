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

### Lets apply the IAM permissions to ASG

Check the attached policy-json file and deploy via aws cli 

```
aws iam put-role-policy --role-name nodes.${KOPS_CLUSTER_NAME} --policy-name asg-nodes.${KOPS_CLUSTER_NAME} --policy-document file:///tmp/autoscale-aws-policy.json
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

### OR Simply deploy the nginx with multiple replica

```
kubectl run nginx --image=nginx --replicas=20
```
- Watch the POD creation and going in pending state
```
nginx-65899c769f-fdkh5   0/1       ContainerCreating   0         2m
nginx-65899c769f-b8xqf   0/1       ContainerCreating   0         2m
nginx-65899c769f-89njs   0/1       Pending   0         0s
nginx-65899c769f-89njs   0/1       Pending   0         0s
nginx-65899c769f-2lpwt   0/1       Pending   0         0s
nginx-65899c769f-2lpwt   0/1       Pending   0         0s
nginx-65899c769f-987tb   0/1       Pending   0         0s
nginx-65899c769f-987tb   0/1       Pending   0         0s

```

- ScaleUP logs you can see in your cluster-scaler pod, max we have only define 2 nodes

```
I1218 06:36:44.219703       1 scale_up.go:199] Best option to resize: db-pool.houm-host-devops-11-sg.k8s.local
I1218 06:36:44.219723       1 scale_up.go:203] Estimated 2 nodes needed in db-pool.houm-host-devops-11-sg.k8s.local
I1218 06:36:44.295078       1 scale_up.go:85] Requested scale-up (2) exceeds node group set capacity, capping to 1
I1218 06:36:44.296184       1 scale_up.go:292] Final scale-up plan: [{db-pool.houm-host-devops-11-sg.k8s.local 1->2 (max: 2)}]
I1218 06:36:44.296197       1 scale_up.go:344] Scale-up: setting group db-pool.houm-host-devops-11-sg.k8s.local size to 2
I1218 06:36:44.426558       1 aws_manager.go:305] Setting asg db-pool.houm-host-devops-11-sg.k8s.local size to 2
I1218 06:36:44.671393       1 factory.go:33] Event(v1.ObjectReference{Kind:"ConfigMap", Namespace:"kube-system", Name:"cluster-autoscaler-stat               us", UID:"0babf809-fc84-11e8-a5f0-02375222ee44", APIVersion:"v1", ResourceVersion:"1882862", FieldPath:""}): type: 'Normal' reason: 'ScaledUpG               roup' Scale-up: group db-pool.houm-host-devops-11-sg.k8s.local size set to 2
I1218 06:36:44.671501       1 factory.go:33] Event(v1.ObjectReference{Kind:"Pod", Namespace:"default", Name:"nginx-65899c769f-wpt8x", UID:"46e               3b762-028f-11e9-a5f0-02375222ee44", APIVersion:"v1", ResourceVersion:"1882921", FieldPath:""}): type: 'Normal' reason: 'TriggeredScaleUp' pod                triggered scale-up: [{db-pool.houm-host-devops-11-sg.k8s.local 1->2 (max: 2)}]
I1218 06:36:44.671529       1 factory.go:33] Event(v1.ObjectReference{Kind:"Pod", Namespace:"default", Name:"nginx-65899c769f-fdkh5", UID:"46e               5d019-028f-11e9-a5f0-02375222ee44", APIVersion:"v1", ResourceVersion:"1882922", FieldPath:""}): type: 'Normal' reason: 'TriggeredScaleUp' pod                triggered scale-up: [{db-pool.houm-host-devops-11-sg.k8s.local 1->2 (max: 2)}]
I1218 06:36:44.671608       1 factory.go:33] Event(v1.ObjectReference{Kind:"Pod", Namespace:"default", Name:"nginx-65899c769f-b8rvv", UID:"46e               073d0-028f-11e9-a5f0-02375222ee44", APIVersion:"v1", ResourceVersion:"1882903", FieldPath:""}): type: 'Normal' reason: 'TriggeredScaleUp' pod                triggered scale-up: [{db-pool.houm-host-devops-11-sg.k8s.local 1->2 (max: 2)}]
I1218 06:36:44.671636       1 factory.go:33] Event(v1.ObjectReference{Kind:"Pod", Namespace:"default", Name:"nginx-65899c769f-b8xqf", UID:"46e               7cb84-028f-11e9-a5f0-02375222ee44", APIVersion:"v1", ResourceVersion:"1882923", FieldPath:""}): type: 'Normal' reason: 'TriggeredScaleUp' pod                triggered scale-up: [{db-pool.houm-host-devops-11-sg.k8s.local 1->2 (max: 2)}]
I1218 06:36:44.671657       1 factory.go:33] Event(v1.ObjectReference{Kind:"Pod", Namespace:"default", Name:"nginx-65899c769f-mpg89", UID:"46e               96556-028f-11e9-a5f0-02375222ee44", APIVersion:"v1", ResourceVersion:"1882925", FieldPath:""}): type: 'Normal' reason: 'TriggeredScaleUp' pod                triggered scale-up: [{db-pool.houm-host-devops-11-sg.k8s.local 1->2 (max: 2)}]
I1218 06:36:44.671678       1 factory.go:33] Event(v1.ObjectReference{Kind:"Pod", Namespace:"default", Name:"nginx-65899c769f-mf6vh", UID:"46e               08e84-028f-11e9-a5f0-02375222ee44", APIVersion:"v1", ResourceVersion:"1882908", FieldPath:""}): type: 'Normal' reason: 'TriggeredScaleUp' pod                triggered scale-up: [{db-pool.houm-host-devops-11-sg.k8s.local 1->2 (max: 2)}]
I1218 06:36:44.671699       1 factory.go:33] Event(v1.ObjectReference{Kind:"Pod", Namespace:"default", Name:"nginx-65899c769f-57s75", UID:"46e               0a4b6-028f-11e9-a5f0-02375222ee44", APIVersion:"v1", ResourceVersion:"1882914", FieldPath:""}): type: 'Normal' reason: 'TriggeredScaleUp' pod                triggered scale-up: [{db-pool.houm-host-devops-11-sg.k8s.local 1->2 (max: 2)}]
I1218 06:36:44.671752       1 factory.go:33] Event(v1.ObjectReference{Kind:"Pod", Namespace:"default", Name:"nginx-65899c769f-h2km8", UID:"46e               0d63b-028f-11e9-a5f0-02375222ee44", APIVersion:"v1", ResourceVersion:"1882918", FieldPath:""}): type: 'Normal' reason: 'TriggeredScaleUp' pod                triggered scale-up: [{db-pool.houm-host-devops-11-sg.k8s.local 1->2 (max: 2)}]
I1218 06:36:44.671773       1 factory.go:33] Event(v1.ObjectReference{Kind:"Pod", Namespace:"default", Name:"nginx-65899c769f-dq7nm", UID:"46e               0c032-028f-11e9-a5f0-02375222ee44", APIVersion:"v1", ResourceVersion:"1882920", FieldPath:""}): type: 'Normal' reason: 'TriggeredScaleUp' pod                triggered scale-up: [{db-pool.houm-host-devops-11-sg.k8s.local 1->2 (max: 2)}]
I1218 06:36:44.671795       1 factory.go:33] Event(v1.ObjectReference{Kind:"Pod", Namespace:"default", Name:"nginx-65899c769f-7nqmp", UID:"46e               82ace-028f-11e9-a5f0-02375222ee44", APIVersion:"v1", ResourceVersion:"1882924", FieldPath:""}): type: 'Normal' reason: 'TriggeredScaleUp' pod                triggered scale-up: [{db-pool.houm-host-devops-11-sg.k8s.local 1->2 (max: 2)}]
I1218 06:36:44.671815       1 factory.go:33] Event(v1.ObjectReference{Kind:"Pod", Namespace:"default", Name:"nginx-65899c769f-m76ls", UID:"46e               9b63f-028f-11e9-a5f0-02375222ee44", APIVersion:"v1", ResourceVersion:"1882926", FieldPath:""}): type: 'Normal' reason: 'TriggeredScaleUp' pod                triggered scale-up: [{db-pool.houm-host-devops-11-sg.k8s.local 1->2 (max: 2)}]
I1218 06:36:44.871315       1 request.go:481] Throttling request took 84.002043ms, request: POST:https://100.64.0.1:443/api/v1/namespaces/defa               ult/events
I1218 06:36:45.070804       1 request.go:481] Throttling request took 196.021356ms, request: POST:https://100.64.0.1:443/api/v1/namespaces/def               ault/events
I1218 06:36:45.270832       1 request.go:481] Throttling request took 107.28453ms, request: POST:https://100.64.0.1:443/api/v1/namespaces/defa               ult/events
I1218 06:36:45.470785       1 request.go:481] Throttling request took 196.312232ms, request: POST:https://100.64.0.1:443/api/v1/namespaces/def               ault/events
I1218 06:36:45.676348       1 request.go:481] Throttling request took 112.698392ms, request: POST:https://100.64.0.1:443/api/v1/namespaces/def               ault/events
I1218 06:36:45.682460       1 leaderelection.go:199] successfully renewed lease kube-system/cluster-autoscaler
I1218 06:36:47.687313       1 leaderelection.go:199] successfully renewed lease kube-system/cluster-autoscaler
I1218 06:36:49.763381       1 leaderelection.go:199] successfully renewed lease kube-system/cluster-autoscaler
I1218 06:36:51.768130       1 leaderelection.go:199] successfully renewed lease kube-system/cluster-autoscaler
I1218 06:36:53.775665       1 leaderelection.go:199] successfully renewed lease kube-system/cluster-autoscaler
I1218 06:36:54.777071       1 static_autoscaler.go:114] Starting main loop
I1218 06:36:55.146177       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I1218 06:36:55.146304       1 static_autoscaler.go:263] Filtering out schedulables
I1218 06:36:55.147060       1 static_autoscaler.go:273] No schedulable pods
I1218 06:36:55.147126       1 scale_up.go:59] Pod default/nginx-65899c769f-mf6vh is unschedulable
I1218 06:36:55.147147       1 scale_up.go:59] Pod default/nginx-65899c769f-57s75 is unschedulable
I1218 06:36:55.147164       1 scale_up.go:59] Pod default/nginx-65899c769f-h2km8 is unschedulable
I1218 06:36:55.147180       1 scale_up.go:59] Pod default/nginx-65899c769f-dq7nm is unschedulable
I1218 06:36:55.147199       1 scale_up.go:59] Pod default/nginx-65899c769f-7nqmp is unschedulable
I1218 06:36:55.147215       1 scale_up.go:59] Pod default/nginx-65899c769f-m76ls is unschedulable
I1218 06:36:55.147242       1 scale_up.go:59] Pod default/nginx-65899c769f-wpt8x is unschedulable
I1218 06:36:55.147296       1 scale_up.go:59] Pod default/nginx-65899c769f-fdkh5 is unschedulable
I1218 06:36:55.147312       1 scale_up.go:59] Pod default/nginx-65899c769f-b8rvv is unschedulable
I1218 06:36:55.147325       1 scale_up.go:59] Pod default/nginx-65899c769f-b8xqf is unschedulable
I1218 06:36:55.147342       1 scale_up.go:59] Pod default/nginx-65899c769f-mpg89 is unschedulable
I1218 06:36:55.320583       1 scale_up.go:92] Upcoming 1 nodes

```

- Verify new node is launched
```
[root@localhost ~]# kubectl get no
NAME                                                STATUS    ROLES     AGE       VERSION
ip-172-20-116-132.ap-southeast-1.compute.internal   Ready     node      12d       v1.10.11
ip-172-20-52-218.ap-southeast-1.compute.internal    Ready     node      7d        v1.10.11
ip-172-20-59-245.ap-southeast-1.compute.internal    Ready     master    12d       v1.10.11
ip-172-20-62-212.ap-southeast-1.compute.internal    Ready     node      5m        v1.10.11
```

### Lets delete deployment, once load is normal then we will see scale down logs and node is removed from cluster and deleted.
```
kubectl delete deploy nginx
kubectl get pod
```
- Lets verify the logs in cluster-scaler pod, one of the node(ip-172-20-62-212.ap-southeast-1.compute.internal) is marked for Scale Down
```
I1218 06:50:38.096240       1 static_autoscaler.go:355] Starting scale down
I1218 06:50:38.226231       1 scale_down.go:387] ip-172-20-62-212.ap-southeast-1.compute.internal was unneeded for 1m3.900298027s
I1218 06:50:38.226348       1 scale_down.go:446] No candidates for scale down
I1218 06:50:40.068016       1 leaderelection.go:199] successfully renewed lease kube-system/cluster-autoscaler
I1218 06:50:42.073734       1 leaderelection.go:199] successfully renewed lease kube-system/cluster-autoscaler
I1218 06:50:44.079463       1 leaderelection.go:199] successfully renewed lease kube-system/cluster-autoscaler
I1218 06:50:46.163581       1 leaderelection.go:199] successfully renewed lease kube-system/cluster-autoscaler
I1218 06:50:48.170029       1 leaderelection.go:199] successfully renewed lease kube-system/cluster-autoscaler
I1218 06:50:48.232620       1 static_autoscaler.go:114] Starting main loop
I1218 06:50:48.466984       1 utils.go:456] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I1218 06:50:48.467068       1 static_autoscaler.go:263] Filtering out schedulables
I1218 06:50:48.467182       1 static_autoscaler.go:273] No schedulable pods
I1218 06:50:48.467209       1 static_autoscaler.go:280] No unschedulable pods
I1218 06:50:48.563280       1 static_autoscaler.go:322] Calculating unneeded nodes
I1218 06:50:48.679678       1 utils.go:404] Skipping ip-172-20-59-245.ap-southeast-1.compute.internal - no node group config
I1218 06:50:48.679790       1 utils.go:404] Skipping ip-172-20-116-132.ap-southeast-1.compute.internal - no node group config
I1218 06:50:48.679935       1 scale_down.go:175] Scale-down calculation: ignoring 1 nodes, that were unremovable in the last 5m0s
I1218 06:50:48.679965       1 scale_down.go:207] Node ip-172-20-62-212.ap-southeast-1.compute.internal - utilization 0.200000
I1218 06:50:48.813362       1 static_autoscaler.go:337] ip-172-20-62-212.ap-southeast-1.compute.internal is unneeded since 2018-12-18 06:49:33.56372828 +0000 UTC duration 1m14.668874519s
I1218 06:50:48.813460       1 static_autoscaler.go:352] Scale down status: unneededOnly=false lastScaleUpTime=2018-12-18 06:36:43.748741588 +0000 UTC lastScaleDownDeleteTime=2018-12-17 16:56:29.117994888 +0000 UTC lastScaleDownFailTime=2018-12-11 06:26:35.921788449 +0000 UTC schedulablePodsPresent=false isDeleteInProgress=false
I1218 06:50:48.813492       1 static_autoscaler.go:355] Starting scale down
I1218 06:50:48.928745       1 scale_down.go:387] ip-172-20-62-212.ap-southeast-1.compute.internal was unneeded for 1m14.668874519s
I1218 06:50:48.928836       1 scale_down.go:446] No candidates for scale down

```

- Lets wait for 10 mins and you see the node is removed, verify same in the cluster-scaler pod
```
1218 06:59:40.770311       1 static_autoscaler.go:355] Starting scale down
I1218 06:59:40.900340       1 scale_down.go:387] ip-172-20-62-212.ap-southeast-1.compute.internal was unneeded for 10m6.673056754s
I1218 06:59:41.025378       1 scale_down.go:594] Scale-down: removing empty node ip-172-20-62-212.ap-southeast-1.compute.internal
I1218 06:59:41.026197       1 factory.go:33] Event(v1.ObjectReference{Kind:"ConfigMap", Namespace:"kube-system", Name:"cluster-autoscaler-status", UID:"0babf809-fc84-11e8-a5f0-02375222ee44", APIVersion:"v1", ResourceVersion:"1886324", FieldPath:""}): type: 'Normal' reason: 'ScaleDownEmpty' Scale-down: removing empty node ip-172-20-62-212.ap-southeast-1.compute.internal
I1218 06:59:41.033643       1 delete.go:53] Successfully added toBeDeletedTaint on node ip-172-20-62-212.ap-southeast-1.compute.internal
I1218 06:59:41.344183       1 aws_manager.go:341] Terminating EC2 instance: i-01fcf5fad5d2844d0
I1218 06:59:41.345184       1 factory.go:33] Event(v1.ObjectReference{Kind:"Node", Namespace:"", Name:"ip-172-20-62-212.ap-southeast-1.compute.internal", UID:"8a20f203-028f-11e9-a5f0-02375222ee44", APIVersion:"v1", ResourceVersion:"1886333", FieldPath:""}): type: 'Normal' reason: 'ScaleDown' node removed by cluster autoscaler

```

- Lets verify the nodes running in cluster,node (ip-172-20-62-212.ap-southeast-1.compute.internal) is removed
```
NAME                                                STATUS    ROLES     AGE       VERSION
ip-172-20-116-132.ap-southeast-1.compute.internal   Ready     node      12d       v1.10.11
ip-172-20-52-218.ap-southeast-1.compute.internal    Ready     node      7d        v1.10.11
ip-172-20-59-245.ap-southeast-1.compute.internal    Ready     master    12d       v1.10.11

```

- TBD
Get the default value checks for scale up
Get the default value time for scale down
How it can be changed

### Reference links
https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler/cloudprovider/aws

https://kumorilabs.com/blog/k8s-5-setup-horizontal-pod-cluster-autoscaling-kubernetes/

https://medium.com/magalix/kubernetes-autoscaling-101-cluster-autoscaler-horizontal-pod-autoscaler-and-vertical-pod-2a441d9ad231

