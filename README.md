# Kubernetes Storage Using Ceph

### What is Kubernetes?
Kubernetes, the awesome container orchestration tool is changing the way applications are being developed and deployed. You can specify the required resources you want and have it available without worrying about the underlying infrastructure. Kubernetes is way ahead in terms of high availability, scaling, managing your application, but storage section in the k8s is still evolving. Many storage supports are getting added and are production-ready.

People are preferring clustered applications to store the data. But, what about the non-clustered applications? Where do these applications store data to make it highly available? Considering these questions, let’s go through the Ceph storage and its integration with Kubernetes.

### What is Ceph Storage?
Ceph is open source, software-defined storage maintained by RedHat. It’s capable of block, object, and file storage. The clusters of Ceph are designed in order to run on any hardware with the help of an algorithm called CRUSH (Controlled Replication Under Scalable Hashing). This algorithm ensures that all the data is properly distributed across the cluster and data quickly without any constraints. Replication, Thin provisioning, Snapshots are the key features of the Ceph storage.

There are good storage solutions like Gluster, Swift but we are going with Ceph for following reasons:
               1. File, Block, and Object storage in the same wrapper.
               2. Better transfer speed and lower latency.
               3. Easily accessible storage that can quickly scale up or down.

We are going to use Ceph-RBD storage in this blog to integrate with Kubernetes.

#### Ceph Deployment
Deploying highly available Ceph cluster is pretty straightforward and easy. I am assuming that you are familiar with setting up the Ceph cluster.

If you check the status, you should see something like:
![alt text](https://github.com/ajinkyaingole30/ceph-manual/blob/master/Screenshot%20(133).png?raw=true)

Here notice that my Ceph monitors hostnames are ip-10-0-0-58, ip-10-0-0-78 and ip-10-0-0-237

There are few things you need to do once your ceph cluster is up and running.

create a pool
``` 
ceph osd pool create kubernetes 1024
```
List pools

```
ceph osd lspools
```
![alt text](https://github.com/ajinkyaingole30/ceph-manual/blob/master/pool.png?raw=true)
Create an RBD_image inside kubernetes pool
```
rbd create kube --size 64 -p kubernetes
```
View created RBD_image
```
rbd info kube -p kubernetes
```
Remove image permissions
```
rbd feature disable kube -p kubernetes object-map fast-diff deep-flatten
```
View created RBD_image and chech if permissions are removed or not
```
rbd info kube -p kubernetes
```

#### K8s Integration
After setting up the Ceph cluster, we would consume it with Kubernetes. I am assuming that your Kubernetes cluster is up and running. We will be using Ceph-RBD as a storage in Kubernetes.
#### Ceph-RBD and Kubernetes
We need a Ceph RBD client to achieve interaction between Kubernetes cluster and CephFS. 
Check the repo https://github.com/ajinkyaingole30/K8s-Ceph.git_

Setting up kubernetes master as ceph client so that we can use rbd_image as storage in Kubernetes

Copy ceph.repo in /etc/yum.repos.d/ and download ceph-common
```
cp ceph.repo /etc/yum.repos.d/ceph.repo
yum install ceph-common
```
start and enable rbdmap service
```
systemctl start rbdmap
systemctl enable rbdmap
```
Copy ceph.client,admin.keyring & also ceph.conf from ceph node and paste it in /etc/ceph of kubernetes-master (ceph client).
you can direct scp it from ceph node to kubernetes master
```
scp /etc/ceph/ceph.client.admin.keyring root@master-node:/etc/ceph/
scp /etc/ceph/ceph.conf root@master-node:/etc/ceph/
```
Check if you can see the pools from kubernetes master (ceph client)
```
rados lspools
```
Now map the rbd_image to your ceph client
```
rbd map kube -p kubernetes
```
check mapped rbd_image
```
rbd showmapped

lsblk
```
This client is not in the official kube-controller-manager container so let’s try to create the external storage plugin for Ceph.

download https://github.com/ajinkyaingole30/K8s-Ceph.git

```
git clone https://github.com/ajinkyaingole30/K8s-Ceph.git
```
Generate a csi-config-map.yaml file similar to the example below, substituting the fsid for “clusterID”, and the monitor addresses for “monitors”:
```
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    [
      {
        "clusterID": "671be3f7-44b4-a79d-07cf4ae58ee1",
        "monitors": [
          "10.0.0.58:6789",
          "10.0.0.78:6789",
          "10.0.0.237:6789"
        ]
      }
    ]
metadata:
  name: ceph-csi-config
```
Once generated, store the new ConfigMap object in Kubernetes:
```
kubectl apply -f csi-config-map.yaml
```
Also generate a ksm-config.yaml
```
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    {
      "vault-test": {
        "encryptionKMSType": "vault",
        "vaultAddress": "http://vault.default.svc.cluster.local:8200",
        "vaultAuthPath": "/v1/auth/kubernetes/login",
        "vaultRole": "csi-kubernetes",
        "vaultPassphraseRoot": "/v1/secret",
        "vaultPassphrasePath": "ceph-csi/",
        "vaultCAVerify": "false"
      }
    }
metadata:
  name: ceph-csi-encryption-kms-config


kubectl apply -f kms-config.yaml
```

ceph-csi requires the cephx credentials for communicating with the Ceph cluster. Generate a csi-rbd-secret.yaml file similar to the example below, using the newly created Kubernetes user id and cephx key:
```
---
apiVersion: v1
kind: Secret
metadata:
  name: csi-rbd-secret
  namespace: default
stringData:
  userID: admin
  userKey: AQD9o0Fd6hQRChAAt7fMaSZXduT3NWEqylNpmg==
```
Once generated, store the new Secret object in Kubernetes:
```
kubectl apply -f csi-rbd-secret.yaml
```
Configure ceph-csi plugins

Create the required ServiceAccount and RBAC ClusterRole/ClusterRoleBinding Kubernetes objects. These objects do not necessarily need to be customized for your Kubernetes environment and therefore can be used as-is from the ceph-csi deployment YAMLs:
```
kubectl apply -f csi-provisioner-rbac.yaml
kubectl apply -f csi-nodeplugin-rbac.yaml
```
Finally, create the ceph-csi provisioner and node plugins. With the possible exception of the ceph-csi container release version, these objects do not necessarily need to be customized for your Kubernetes environment and therefore can be used as-is from the ceph-csi deployment YAMLs:
```
kubectl apply -f csi-rbdplugin-provisioner.yaml
kubectl apply -f csi-rbdplugin.yaml
```
Using Ceph Block Devices create a storageclass

The Kubernetes StorageClass defines a class of storage. Multiple StorageClass objects can be created to map to different quality-of-service levels (i.e. NVMe vs HDD-based pools) and features.

For example, to create a ceph-csi StorageClass that maps to the kubernetes pool created above, the following YAML file can be used after ensuring that the “clusterID” property matches your Ceph cluster’s fsid:

```
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: csi-rbd-sc
provisioner: rbd.csi.ceph.com
parameters:
   clusterID: 671be3f7-44b4-a79d-07cf4ae58ee1
   pool: kubernetes
   csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
   csi.storage.k8s.io/provisioner-secret-namespace: default
   csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
   csi.storage.k8s.io/node-stage-secret-namespace: default
reclaimPolicy: Delete
mountOptions:
   - discard

kubectl apply -f csi-rbd-sc.yaml
```

Create a Persistent Volume Claim

A PersistentVolumeClaim is a request for abstract storage resources by a user. The PersistentVolumeClaim would then be associated to a Pod resource to provision a PersistentVolume, which would be backed by a Ceph block image. An optional volumeMode can be included to select between a mounted file system (default) or raw block device-based volume.

To create a block-based PersistentVolumeClaim that utilizes the ceph-csi-based StorageClass created above, the following YAML can be used to request raw block storage from the csi-rbd-sc StorageClass:
```
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: raw-block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 50Mi
  storageClassName: csi-rbd-sc

kubectl apply -f raw-block-pvc.yaml
```
Check the pod if it is running or not
```
kubectl get pods
```
![alt text](https://github.com/ajinkyaingole30/ceph-manual/blob/master/1.pod.png?raw=true)
Check if the pvc is bound
```
kubectl get pvc
```
Check if the pv is bound
```
kubectl get pv
```
![alt text](https://github.com/ajinkyaingole30/ceph-manual/blob/master/1.pv.png?raw=true)
To create a file-system-based PersistentVolumeClaim that utilizes the ceph-csi-based StorageClass created above, the following YAML can be used to request a mounted file system (backed by an RBD image) from the csi-rbd-sc StorageClass:
```
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 50Mi
  storageClassName: csi-rbd-sc


kubectl apply -f fs_pvc.yaml
```
Check if the pvc rbd-pvc is bound or not
```
kubectl get pvc
```
The following demonstrates and example of binding the above PersistentVolumeClaim to a Pod resource as a mounted file system:
```
---
apiVersion: v1
kind: Pod
metadata:
  name: ceph-pod
spec:
  containers:
    - name: web-server
      image: nginx
      volumeMounts:
        - name: mypvc
          mountPath: /var/lib/www/html
  volumes:
    - name: mypvc
      persistentVolumeClaim:
        claimName: rbd-pvc
        readOnly: false

kubectl apply -f pod.yaml
```
Check the pod if it is running or not
```
kubectl get pods
```
![alt text](https://github.com/ajinkyaingole30/ceph-manual/blob/master/1.pod.png?raw=true)
Get into the pod to check if the volume is attached
```
kubectl exec -it ceph-pod -- df -hT | grep /dev/rbd0
```
![alt text](https://github.com/ajinkyaingole30/ceph-manual/blob/master/1.podexec.png?raw=true)

Using this you can integrate ceph with kubernetes___
![alt text](https://github.com/ajinkyaingole30/ceph-manual/blob/master/1.exec.png?raw=true)

I hope you enjoyed the article and found the information useful. Please share your feedback.

### Happy Cephing!
