# Horizontal Pod Autoscaler - HPA
As the name implies, HPA scales the number of pod replicas. Most DevOps use CPU and memory as the triggers to scale more pod replicas or less.

## High level HPA workflow

![HPA WorkFlow](https://github.com/sanjaynaikwadi/kubernetes/blob/master/AutoScaling/HPA/HPA.png)

### Prerequisites

- Kubernetes Cluster (I tested with 1.10.11 via KOPS on AWS)
- Metric Server installed
![How to Setup Metric Server]()


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

### Lets Play 
- Lets deploy the php app for testing
```
$ kubectl run php-apache --image=k8s.gcr.io/hpa-example --requests=cpu=200m --expose --port=80
service/php-apache created
deployment.apps/php-apache created
```
Once your pod is up and in running mode we need to add that deployment to autoscale, before proceeding verify your php-apache is running.


