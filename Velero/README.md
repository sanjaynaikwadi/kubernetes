## Overview

Velero (formerly Heptio Ark) gives you tools to back up and restore your Kubernetes cluster resources and persistent volumes. You can run Velero with a public cloud platform or on-premises. Velero lets you:
```
   Take backups of your cluster and restore in case of loss.
   Migrate cluster resources to other clusters.
   Replicate your production cluster to development and testing clusters.
```
Velero consists of:
```
   A server that runs on your cluster
   A command-line client that runs locally
```

## On-premises 
I have K8s cluster deployed in my local datacenter enviornment, with one Master and 3 worker nodes.
For Persistence Volumes I am us Portworx as storage solution, you will find lot of storage solution available in market. Portworx is paid solution but they too have Freemium product
 as PX-Essentials which meets requirements for smaller clusters. There are other opensource soultions like OpenEBS, Rook, Robin etc. It's your choice what suits you best.

As setup is on-prem you would need an S3 base object store for taking backups. We will be using as `Minio` which is S3 compitable object store which is widely used.

## Perquisites
* K8s installed (1.15 Tested)
* Portworx PX-Essential (or Enterprise) installed 
* Make sure your Portworx is up and running 


## Install Minio
There are multiple ways of installing Minio
* StatefulSet Spec
* Helm
* Operator 

More info can be found on : https://docs.min.io/docs/deploy-minio-on-kubernetes.html

- Lets Create the StorageClass & Statefulset
vi minio-sc.yaml
```yaml

kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: minio-sc
provisioner: kubernetes.io/portworx-volume
parameters:
  repl: "2"
```

`kubectl apply -f minio-sc.yaml`




