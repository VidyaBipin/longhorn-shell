# Longhorn

Longhorn is a distributed block storage system for Kubernetes. Longhorn is lightweight, reliable, and easy-to-use. You can deploy Longhorn on an existing Kubernetes cluster with one simple command. Once Longhorn is deployed, it adds persistent volume support to the Kubernetes cluster.

Longhorn implements distributed block storage using containers and microservices. Longhorn creates a dedicated storage controller for each block device volume and sychronously replicates the volume across multiple replicas stored on multiple nodes. The storage controller and replicas are themselves orchestrated using Kubernetes. Longhorn supports snapshots, backups, and even allows you to schedule recurring snapshots and backups!

You can read more details of Longhorn and its design [here](http://rancher.com/microservices-block-storage/).

Longhorn is a work in progress. We appreciate your comments as we continue to work on it!

## Source Code
Longhorn is 100% open source software. Project source code is spread across a number of repos:

1. Longhorn Engine -- Core controller/replica logic https://github.com/rancher/longhorn-engine
1. Longhorn Manager -- Longhorn orchestration, includes Flexvolume driver for Kubernetes https://github.com/rancher/longhorn-manager
1. Longhorn UI -- Dashboard https://github.com/rancher/longhorn-ui

# Deploy on Kubernetes

## Requirements

1. Docker v1.13+
2. Kubernetes v1.8+
3. Make sure `curl`, `findmnt`, `grep`, `awk` and `blkid` has been installed in all nodes of the Kubernetes cluster.
4. Make sure `open-iscsi` has been installed in all nodes of the Kubernetes cluster. For GKE, recommended Ubuntu as guest OS image since it contains `open-iscsi` already.

## Deployment
Create the deployment of Longhorn in your Kubernetes cluster is easy. For most Kubernetes setup (except GKE), you will only need to run the following command to install Longhorn:
```
kubectl create -f https://raw.githubusercontent.com/rancher/longhorn/0.2/deploy/longhorn.yaml
```

For Google Kubernetes Engine (GKE) users, see [here](#google-kubernetes-engine) before proceed.

Longhorn Manager and Longhorn Driver will be deployed as daemonsets in a separate namespace called `longhorn-system`, as you can see in the yaml file.

When you see those pods has started correctly as follows, you've deployed the Longhorn successfully.

```
# kubectl -n longhorn-system get pod
NAME                                                  READY     STATUS    RESTARTS   AGE
longhorn-flexvolume-driver-4dnx6                      1/1       Running   0          1d
longhorn-flexvolume-driver-cqwj5                      1/1       Running   0          1d
longhorn-flexvolume-driver-deployer-bc7b95b5b-sb9kr   1/1       Running   0          1d
longhorn-flexvolume-driver-q9h4f                      1/1       Running   0          1d
longhorn-manager-dkdn9                                1/1       Running   0          2h
longhorn-manager-l6npd                                1/1       Running   0          2h
longhorn-manager-v4fz8                                1/1       Running   0          2h
longhorn-ui-58796c68d-db4t6                           1/1       Running   0          1h
```

## Access the UI
Use `kubectl -n longhorn-system get svc` to get the external service IP for UI:

```
NAME                TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
longhorn-backend    ClusterIP      10.20.248.250   <none>           9500/TCP       58m
longhorn-frontend   LoadBalancer   10.20.245.110   100.200.200.123   80:30697/TCP   58m
```

Then user can use `EXTERNAL-IP`(`100.200.200.123` in the case above) of `longhorn-frontend` to access the Longhorn UI.

##  How to use the Longhorn Volume in your pod

There are serveral ways to use the Longhorn volume.

### Pod with Longhorn volume
The following YAML file shows the definition of a pod that makes the Longhorn attach a volume to be used by the pod.

```
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
  namespace: default
spec:
  containers:
  - name: volume-test
    image: nginx:stable-alpine
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: voll
      mountPath: /data
    ports:
    - containerPort: 80
  volumes:
  - name: voll
    flexVolume:
      driver: "rancher.io/longhorn"
      fsType: "ext4"
      options:
        size: "2Gi"
        numberOfReplicas: "3"
        staleReplicaTimeout: "20"
        fromBackup: ""
```

Notice this field in YAML file `flexVolume.driver "rancher.io/longhorn"`. It specifies Longhorn FlexVolume plug-in shoule be used. There are some options fields in `options` user can fill.

Option  | Required | Description
------------- | ----|---------
size    |  Yes | Specify the capacity of the volume in longhorn and the unit should be `G`
numberOfReplicas | Yes | The number of replica (HA feature) for volume in this Longhorn volume
fromBackup | No | Optional. Must be a Longhorn Backup URL. Specify where user want to restore the volume from.

### Storage class

Longhorn supports dynamic provisioner function, which can create PV automatically for the user according to the spec of storage class and PVC. User need to create a new storage class in order to use it. The storage class example is at [here](./deploy/example-storageclass.yaml)
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: longhorn
provisioner: rancher.io/longhorn
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "30"
  fromBackup: ""
```

Then user can create PVC directly. For example:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-volv-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 2Gi
```

THen use it in the pod:
```
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
  namespace: default
spec:
  containers:
  - name: volume-test
    image: nginx:stable-alpine
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: volv
      mountPath: /data
    ports:
    - containerPort: 80
  volumes:
  - name: volv
    persistentVolumeClaim:
      claimName: longhorn-volv-pvc
```

## Setup a TESTING ONLY NFS server for storing backups

Longhorn supports backing up mechanism to export the user data out of Longhorn system. Currently Longhorn supports backing up to a NFS server. In order to use this feature, you need to have a NFS server running and accessible in the Kubernetes cluster. Here we provides a simple way help to setup a testing NFS server.

WARNING: This NFS server won't save any data after you delete it. It's for TESTING ONLY.

```
kubectl create -f deploy/example-backupstore.yaml
```
It will create a simple NFS server in the `default` namespace, which can be addressed as `longhorn-test-nfs-svc.default` for other pods in the cluster.

After this script completes, using the following URL as the Backup Target in the Longhorn setting:
```
nfs://longhorn-test-nfs-svc.default:/opt/backupstore
```
Open Longhorn UI, go to Setting, fill the Backup Target field with the URL above, click Save. Now you should able to use the backup feature of Longhorn.

## Google Kubernetes Engine
The configuration yaml will be slight different for Google Kubernetes Engine (GKE):

1. GKE requires user to manually claim himself as cluster admin to enable RBAC. User need to execute following command before create the Longhorn system using yaml files.
```
kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=<name@example.com>
```
In which `name@example.com` is the user's account name in GCE, and it's case sensitive.
See [here](https://cloud.google.com/kubernetes-engine/docs/how-to/role-based-access-control) for details.

2. The default Flexvolume plugin directory is different with GKE 1.8+, which is at `/home/kubernetes/flexvolume`. User need to use following command instead:
```
kubectl create -f https://raw.githubusercontent.com/rancher/longhorn/0.2/deploy/longhorn-gke.yaml
```

User can also customerize the Flexvolume directory in the last part of the Longhorn system deployment yaml file, e.g.:
```
          - name: FLEXVOLUME_DIR
            value: "/home/kubernetes/flexvolume/"
```

See [Troubleshooting](#troubleshooting) for details.

## Uninstall Longhorn

Longhorn can be easily uninstalled using:
```
kubectl delete -f https://raw.githubusercontent.com/rancher/longhorn/0.2/deploy/longhorn.yaml
```

## Troubleshooting

### Volume can be attached/detached from UI, but Kubernetes Pod/StatefulSet etc cannot use it

Check if volume plugin directory has been set correctly.

By default, Kubernetes use `/usr/libexec/kubernetes/kubelet-plugins/volume/exec/` as the directory for volume plugin drivers, as stated in the [official document](https://github.com/kubernetes/community/blob/master/contributors/devel/flexvolume.md#prerequisites).

But some vendors may choose to change the directory due to various reasons. For example, GKE uses `/home/kubernetes/flexvolume` instead.

User can find the correct directory by running `ps aux|grep kubelet` on the host and check the `--volume-plugin-dir` parameter. If there is none, the default `/usr/libexec/kubernetes/kubelet-plugins/volume/exec/` will be used.

## License
Copyright (c) 2014-2018 [Rancher Labs, Inc.](http://rancher.com)

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

[http://www.apache.org/licenses/LICENSE-2.0](http://www.apache.org/licenses/LICENSE-2.0)

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
