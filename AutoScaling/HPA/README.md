# Horizontal Pod Autoscaler - HPA
As the name implies, HPA scales the number of pod replicas. Most DevOps use CPU and memory as the triggers to scale more pod replicas or less.

## High level HPA workflow

[HPA WorkFlow](https://github.com/sanjaynaikwadi/kubernetes/blob/master/AutoScaling/HPA/HPA.png)

### Prerequisites

- Kubernetes Cluster (I tested with 1.10.11 via KOPS on AWS)
- Resource limit set in deployment (check the default yaml file attached for reference)
- Metric Server installed

[How to Setup Metric Server](https://github.com/sanjaynaikwadi/kubernetes/tree/master/AutoScaling/Metric-Server)


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

### Lets Play with CPU scaling   
- Lets deploy the php app for testing which will be scaled on base of cpu metrics
```
$ kubectl run php-apache --image=k8s.gcr.io/hpa-example --requests=cpu=200m --expose --port=80
service/php-apache created
deployment.apps/php-apache created
```

### Create Horizontal Pod Autoscaler
The following command will create a Horizontal Pod Autoscaler that maintains between 1 and 10 replicas of the Pods controlled by the php-apache deployment.
```
$ kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
horizontalpodautoscaler.autoscaling/php-apache autoscaled
```

We may check the current status of autoscaler by running:
```
$ kubectl get hpa
NAME         REFERENCE                     TARGET    MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache/scale   0% / 50%  1         10        1          18s
```

### Lets Add the Load
We will start a container, and send an infinite loop of queries to the php-apache service (please run it in a different terminal):
```
$ kubectl run -i --tty load-generator --image=busybox /bin/sh

Hit enter for command prompt

$ while true; do wget -q -O- http://php-apache.default.svc.cluster.local; done
```

Within a minute or so, we should see the higher CPU load by executing:
```
$ kubectl get hpa
NAME         REFERENCE                     TARGET      CURRENT   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache/scale   305% / 50%  305%      1         10        1          3m
```

Here, CPU consumption has increased to 305% of the request. As a result, the deployment was resized to 7 replicas:
```
$ kubectl get deployment php-apache
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
php-apache   7         7         7            7           19m
```

### Stop the load
In the terminal where we created the container with busybox image, terminate the load generation by typing <Ctrl> + C
Wait for a minute or so and lets verify the status again
```
$ kubectl get hpa
NAME         REFERENCE                     TARGET       MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache/scale   0% / 50%     1         10        1          11m

$ kubectl get deployment php-apache
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
php-apache   1         1         1            1           27m
```

Cool Your done, you can scale your pods according to your workload.

### Reference Links for additional information
https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/

https://github.com/kubernetes/community/blob/master/contributors/design-proposals/autoscaling/horizontal-pod-autoscaler.md#autoscaling-algorithm

https://medium.com/magalix/kubernetes-autoscaling-101-cluster-autoscaler-horizontal-pod-autoscaler-and-vertical-pod-2a441d9ad231




