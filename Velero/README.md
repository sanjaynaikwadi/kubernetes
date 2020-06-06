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

- Create the StorageClass

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

- Create Headless Service

vi minio-headless-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: minio
  labels:
    app: minio
spec:
  clusterIP: None
  ports:
    - port: 9000
      name: minio
  selector:
    app: minio
```

`kubectl apply -f minio-headless-service.yaml`

- Create the StatefulSet

vi minio-sts.yaml

```yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: minio
spec:
  serviceName: minio
  replicas: 4
  template:
    metadata:
      annotations:
        pod.alpha.kubernetes.io/initialized: "true"
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        env:
        - name: MINIO_ACCESS_KEY
          value: "minio"
        - name: MINIO_SECRET_KEY
          value: "minio123"
        image: minio/minio:RELEASE.2020-04-28T23-56-56Z
        args:
        - server
        - http://minio-0.minio.default.svc.cluster.local/data
        - http://minio-1.minio.default.svc.cluster.local/data
        - http://minio-2.minio.default.svc.cluster.local/data
        - http://minio-3.minio.default.svc.cluster.local/data
        ports:
        - containerPort: 9000
          hostPort: 9000
        # These volume mounts are persistent. Each pod in the PetSet
        # gets a volume mounted based on this field.
        volumeMounts:
        - name: data
          mountPath: /data
  # These are converted to volume claims by the controller
  # and mounted at the paths mentioned above.
  volumeClaimTemplates:
  - metadata:
      name: data
      annotations:
        volume.beta.kubernetes.io/storage-class: minio-sc
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi
```

- Expose the Service via NodePort or LoadBalancer

vi minio-service-expose.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: minio-service
spec:
  type: NodePort
  ports:
    - port: 9000
      nodePort: 30000
  selector:
    app: minio
```

`kubectl apply -f minio-service-expose.yaml`
`

- Access Minio

