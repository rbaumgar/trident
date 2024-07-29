# Using FSx ONTAP Storage for OpenShift Virtualization on ROSA

See Using FSx ONTAP Storage Provider for OpenShift Virtualization on AWS Bare Metal, https://access.redhat.com/articles/7057644

## create "OpenShift Virtualization on ROSA Demo"

at demo.redhat.com

get AWS credentials from demo env and login.

## create FSx

got to
https://console.aws.amazon.com/fsx/home [check correct region!]


```
select: Amazon FSx for NetApp ONTAP
- click "next"
Quick create
File system name: cnv-fsx
Deployment type: Single-AZ
SSD: 1024 [GB]
Virtual Private Cloud: rosa-.....-vpc
Storage efficiency: enabled (recommended)
- click "Create file system"
```

Will take 30-45 minutes.

When finished
- delete vol: at fsx > File systems, select Volumes, delete vol
- change backups to no: at fsx > File systems, select Backups, settings: update, Enable automatic backups: no. Delete all backups

## Edit SecurityGroup DefaultSG
!!! Network Security !!!

go to
https://console.aws.amazon.com/ec2/home?#SecurityGroups:


Edit "DefaultSG", remove inbound rules, update outbound rules to "0.0.0.0/16"

## create Astra Trident Operator, 24.6

Examples Create your first NFS backends for Trident & Storage Classes for Kubernetes, https://github.com/YvosOnTheHub/LabNetApp/tree/master/Kubernetes_v5/Scenarios/Scenario02

Deploy *Astra Trident Operator* on the OpenShift console.

## Create Trident Orchestrator

```
apiVersion: trident.netapp.io/v1
kind: TridentOrchestrator
metadata:
  name: trident
spec:
  imagePullSecrets: []
  IPv6: false
  enableNodePrep: false
  debug: false
  imageRegistry: ''
  namespace: trident
  k8sTimeout: 30
  kubeletDir: /var/lib/kubelet
  silenceAutosupport: false
```

## Create Trident Backend Config

dataLIF and managementLIF can be found at Storage virtual machines, Endpoints, Management DNS name


```
apiVersion: trident.netapp.io/v1
kind: TridentBackendConfig
metadata:
  name: backend-tbc-ontap-nas-default
  namespace: trident
spec:
  dataLIF: svm-061ca9ec6682138b1.fs-0ed8a538548a33628.fsx.eu-west-1.amazonaws.com
  managementLIF: svm-061ca9ec6682138b1.fs-0ed8a538548a33628.fsx.eu-west-1.amazonaws.com
  storageDriverName: ontap-nas
  autoExportPolicy: true
  version: 1
  credentials:
    name: vsadmin-user
  autoExportCIDRs:
    - 10.0.0.0/16
  backendName: nas-default
```

Annotation: useRest=true slow!

## SVM Admin Username/Password

SVMAdmin username and password can be found at Storage virtual machines, Administration, Update!


```
apiVersion: v1
kind: Secret
metadata:
  name: vsadmin-user
type: Opaque
stringData:
  username: vsadmin
  password: SVMAdmin1
```

## Create Storage Class

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: trident-csi-fsx
provisioner: csi.trident.netapp.io
parameters:
  backendType: ontap-nas
  csi.storage.k8s.io/fstype: “nfs”
allowVolumeExpansion: true
volumeBindingMode: Immediate
```

Attantion: If you have Multiple TridentBackendConfigs add storagePool, eg "cnv = backendName

```
  storagePools: 'cnv:.*'
```

## Create a demo PVC

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: aaaa
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
  storageClassName: trident-csi-fsx
```


## (optional) make default storage class!

```
add annotation to the storageClass:
  annotations:
    storageclass.kubernetes.io/is-default-class: 'true'
```

## (debug) Trident controller logs

```
oc logs deploy/trident-controller -n trident
```

If you need more details, update TridentOrchestrator debug:true.

## Lab Guide

https://redhat-gpst.github.io/rosa-fsx-lab-guide/modules/index.html