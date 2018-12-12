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

