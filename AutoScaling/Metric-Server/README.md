# Kubernetes Metric Server
Starting from Kubernetes 1.8, resource usage metrics, such as container CPU and memory usage, are available in Kubernetes through the Metrics API.
Metric server collects metrics from the Summary API, exposed by Kubelet on each node.

### Pre-requiste 
- Kubernetes Cluster 1.8+

### Installing
- Clone the repo
- Deploy all yaml files, run the below command where all yaml files reside
```
for yaml in `ls *.yaml`; do kubectl create -f $yaml; done
```
- I have modified the yaml file as I was having issue while deploying the default one, below is the content added to metrics-server-deployment-modified.yaml
```
        imagePullPolicy: Always
        command:
        - /metrics-server
        - --kubelet-insecure-tls
        - --kubelet-preferred-address-types=InternalIP
```
### Verify
- Verify the pod is running
```
[root@localhost Metric-Server]# kubectl get po -n kube-system -l "k8s-app=metrics-server"
NAME                             READY     STATUS    RESTARTS   AGE
metrics-server-98bf7d47b-wjkl5   1/1       Running   0          5d
```

- Verify the output of top node or pod
```
[root@localhost Metric-Server]# kubectl top no
NAME                                                CPU(cores)   CPU%      MEMORY(bytes)   MEMORY%
ip-172-20-116-132.ap-southeast-1.compute.internal   20m          2%        568Mi           63%
ip-172-20-52-218.ap-southeast-1.compute.internal    20m          2%        476Mi           53%
ip-172-20-59-245.ap-southeast-1.compute.internal    112m         5%        3349Mi          42%
```

Cool you set with getting the metrics

### Reference links
https://kubernetes.io/docs/tasks/debug-application-cluster/core-metrics-pipeline/

https://github.com/kubernetes-incubator/metrics-server




