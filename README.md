# k8s-ceph

# Setup

## Install k8s with calico network
 - [Setup kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)
 - [Setup k8s](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)

## Install ceph
 - [Single node installation](http://palmerville.github.io/2016/04/30/single-node-ceph-install.html) - params for single node and fixe permission on *keyring file
 - [Installing specific version of ceph](http://docs.ceph.com/docs/master/rados/deployment/ceph-deploy-install/#install)
 - [Official installation guide](http://docs.ceph.com/docs/master/start/quick-ceph-deploy/)
 - [Ceph cheatsheet](http://rebirther.ru/blog/ceph-spargalka)

## k8s + ceph

#### Working links

 - [bug zilla](https://bugzilla.redhat.com/show_bug.cgi?id=1460275) 
 befort create ceph image run this command on ceph-mon node
```bash
rbd create --image ceph-image --size 2G --image-feature layering
```
 - [example](https://access.redhat.com/documentation/en-us/openshift_container_platform/3.5/html-single/installation_and_configuration/#using-ceph-rbd-installing-the-ceph-common-package)

#### Installing the ceph-common

Installing the ceph-common Package The ceph-common library must be installed on all schedulable OpenShift Container Platform nodes: 
```bash
yum install -y ceph-common
```

#### Creating the Ceph Secret 
The **ceph auth get-key** command is run on a Ceph **MON** node to display the key value for the **client.admin user**: 

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
data:
  key: QVFBOFF2SlZheUJQRVJBQWgvS2cwT1laQUhPQno3akZwekxxdGc9PQ== #1
```
**1** 
This base64 key is generated on one of the Ceph MON nodes using the **ceph auth get-key client.admin | base64** command, then copying the output and pasting it as the secret key’s value. 

Save the secret definition to a file, for example ceph-secret.yaml, then create the secret:
```bash
$ oc create -f ceph-secret.yaml
secret "ceph-secret" created
```

Verify that the secret was created: 
```bash
# oc get secret ceph-secret
NAME          TYPE      DATA      AGE
ceph-secret   Opaque    1         23d
```
#### Creating the Persistent Volume 

Next, before creating the PV object in OpenShift Container Platform, define the persistent volume file: 
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ceph-pv     #1
spec:
  capacity:
    storage: 2Gi    #2
  accessModes:
    - ReadWriteOnce #3
  rbd:              #4
    monitors:       #5
      - 192.168.122.133:6789
    pool: rbd
    image: ceph-image
    user: admin
    secretRef:
      name: ceph-secret #6
    fsType: ext4        #7
    readOnly: false
  persistentVolumeReclaimPolicy: Recycle
```

**1**
    The name of the PV, which is referenced in pod definitions or displayed in various oc volume commands. 
**2**
    The amount of storage allocated to this volume. 
**3**
    accessModes are used as labels to match a PV and a PVC. They currently do not define any form of access control. All block storage is defined to be single user (non-shared storage). 
**4**
    This defines the volume type being used. In this case, the rbd plug-in is defined. 
**5**
    This is an array of Ceph monitor IP addresses and ports. 
**6**
    This is the Ceph secret, defined above. It is used to create a secure connection from OpenShift Container Platform to the Ceph server. 
**7**
    This is the file system type mounted on the Ceph RBD block device.

Save the PV definition to a file, for example ceph-pv.yaml, and create the persistent volume: 
```bash
# oc create -f ceph-pv.yaml
persistentvolume "ceph-pv" created
```

Verify that the persistent volume was created: 
```bash
# oc get pv
NAME                     LABELS    CAPACITY     ACCESSMODES   STATUS      CLAIM     REASON    AGE
ceph-pv                  <none>    2147483648   RWO           Available                       2s
```

####  Creating the Persistent Volume Claim

Creating the Persistent Volume Claim A persistent volume claim (PVC) specifies the desired access mode and storage capacity. Currently, based on only these two attributes, a PVC is bound to a single PV. Once a PV is bound to a PVC, that PV is essentially tied to the PVC’s project and cannot be bound to by another PVC. There is a one-to-one mapping of PVs and PVCs. However, multiple pods in the same project can use the same PVC. 
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ceph-claim
spec:
  accessModes:     #1
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi #2
```
**1**
    As mentioned above for PVs, the accessModes do not enforce access right, but rather act as labels to match a PV to a PVC. 
**2**
    This claim will look for PVs offering 2Gi or greater capacity. 

Save the PVC definition to a file, for example ceph-claim.yaml, and create the PVC: 

```bash
# oc create -f ceph-claim.yaml
persistentvolumeclaim "ceph-claim" created

#and verify the PVC was created and bound to the expected PV:
# oc get pvc
NAME         LABELS    STATUS    VOLUME    CAPACITY   ACCESSMODES   AGE
ceph-claim   <none>    Bound     ceph-pv   1Gi        RWX           21s
```

#### Creating the Pod
Creating the Pod A pod definition file or a template file can be used to define a pod. Below is a pod specification that creates a single container and mounts the Ceph RBD volume for read-write access: 
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ceph-pod1           #1
spec:
  containers:
  - name: ceph-busybox
    image: busybox          #2
    command: ["sleep", "60000"]
    volumeMounts:
    - name: ceph-vol1       #3
      mountPath: /usr/share/busybox #4
      readOnly: false
  volumes:
  - name: ceph-vol1         #5
    persistentVolumeClaim:
      claimName: ceph-claim #6
```
**1**
    The name of this pod as displayed by oc get pod. 
**2**
    The image run by this pod. In this case, we are telling busybox to sleep. 
**3** **5**
    The name of the volume. This name must be the same in both the containers and volumes sections. 
**4**
    The mount path as seen in the container. 
**6**
    The PVC that is bound to the Ceph RBD cluster. 

Save the pod definition to a file, for example ceph-pod1.yaml, and create the pod: 

```bash
# oc create -f ceph-pod1.yaml
pod "ceph-pod1" created

#verify pod was created
# oc get pod
NAME        READY     STATUS    RESTARTS   AGE
ceph-pod1   1/1       Running   0          2m
```