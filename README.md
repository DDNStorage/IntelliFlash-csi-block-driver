# IntelliFlash CSI Block Driver

## Overview
The Intelliflash Container Storage Interface (CSI) Block Driver provides a CSI interface used by Container Orchestrators (CO) to manage the lifecycle of Intelliflash volumes over iSCSI protocol.

## Feature List
|Feature|Feature Status|CSI Driver Version|CSI Spec Version|Kubernetes Version|Intelliflash Version|
|--- |--- |--- |--- |--- |--- |
|Static Provisioning|GA|>= v1.0.0|>= v1.0.0|>=1.13|>=3.11.2|
|Dynamic Provisioning|GA|>= v1.0.0|>= v1.0.0|>=1.13|>=3.11.2|
|RW mode|GA|>= v1.0.0|>= v1.0.0|>=1.13|>=3.11.2|
|RO mode|GA|>= v1.0.0|>= v1.0.0|>=1.13|>=3.11.2|
|Creating and deleting snapshot|GA|>= v1.2.0|>= v1.0.0|>=1.20|>=3.11.2|
|Provision volume from snapshot|GA|>= v1.2.0|>= v1.0.0|>=1.20|>=3.11.2|
|Provision volume from another volume|GA|>= v1.3.0|>= v1.0.0|>=1.20|>=3.11.2|
|List snapshots of a volume|Beta|>= v1.2.0|>= v1.0.0|>=1.20|>=3.11.2|
|Expand volume|GA|>= v1.3.0|>= v1.1.0|>=1.16|>=3.11.2|
|Access list for volume (NFS only)|GA|>= v1.3.0|>= v1.0.0|>=1.13|>=3.11.2|
|Topology|Beta|>= v1.4.0|>= v1.0.0|>=1.17|>=3.11.2|
|Raw block device|In development|future|>= v1.0.0|>=1.14|>=3.11.2|
|StorageClass Secrets|Beta|>= v1.3.0|>=1.0.0|>=1.13|>=3.11.2|
|Mount options|GA|>=v1.0.0|>=v1.0.0|>=v1.13|>=3.11.2|

## Access Modes support
|Access mode| Supported in version|
|--- |--- |
|ReadWriteOnce| >=1.0.0 |
|ReadOnlyMany| >=1.0.0 |
|ReadWriteMany| >=1.0.0 |
|ReadWriteOncePod| >=1.1.0 |

## Requirements

- Kubernetes cluster must allow privileged pods, this flag must be set for the API server and the kubelet
  ([instructions](https://github.com/kubernetes-csi/docs/blob/735f1ef4adfcb157afce47c64d750b71012c8151/book/src/Setup.md#enable-privileged-pods)):
  ```
  --allow-privileged=true
  ```
- Required the API server and the kubelet feature gates
  ([instructions](https://github.com/kubernetes-csi/docs/blob/735f1ef4adfcb157afce47c64d750b71012c8151/book/src/Setup.md#enabling-features)):
  ```
  --feature-gates=VolumeSnapshotDataSource=true,VolumePVCDataSource=true,ExpandInUsePersistentVolumes=true,ExpandCSIVolumes=true,ExpandPersistentVolumes=true,BlockVolume=true,CSIBlockVolume=true
  ```
  If you are planning on using topology, the following feature-gates are required
  ```
  ServiceTopology=true,CSINodeInfo=true,Topology=true,
  ```
- Mount propagation must be enabled, the Docker daemon for the cluster must allow shared mounts
  ([instructions](https://github.com/kubernetes-csi/docs/blob/735f1ef4adfcb157afce47c64d750b71012c8151/book/src/Setup.md#enabling-mount-propagation))

  ## Installation

1. Create Intelliflash project for the driver, example: `csi-block`.

2. Clone driver repository
   ```bash
   git clone https://bitbucket.eng-us.tegile.com/eco/intelliflash-csi-block-driver.git
   cd intelliflash-csi-block-driver
   ```
3. Edit `deploy/kubernetes/intelliflash-csi-block-driver-config.yaml` file. Driver configuration example:

```yaml
arrays:
  array1:
    restIp: https://172.27.10.30:443   # [required] IntelliFlash REST API endpoint
    username: admin                    # [required] IntelliFlash REST API username
    password: t                        # [required] IntelliFlash REST API password
    defaultProject: csi-block          # default project name for driver's volume
    defaultDataIp: 172.27.10.30        # default IntelliFlash data IP

useChapAuth: true                           # Defines whether CHAP authentication is enabled
chapUser: admin                             # CHAP username
chapSecret: chapsecretif                    # CHAP secret
initiatorGroup: csi-block-chap-initiator-group  # Initiator group associated with project, required for CHAP
debug: true
```

4. Create Kubernetes secret from the file:
   ```bash
   kubectl create secret generic intelliflash-csi-block-driver-config --from-file=deploy/kubernetes/intelliflash-csi-block-driver-config.yaml
   ```
5. Register driver to Kubernetes:
   ```bash
   kubectl apply -f deploy/kubernetes/intelliflash-csi-block-driver.yaml
   ```

## Usage

### Dynamically provisioned volumes

For dynamic volume provisioning, the administrator needs to set up a _StorageClass_ pointing to the driver.
In this case Kubernetes generates volume name automatically (for example `pvc-ns-cfc67950-fe3c-11e8-a3ca-005056b857f8`).
Default driver configuration may be overwritten in `parameters` section:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: intelliflash-csi-block-driver-sc-nginx-dynamic
provisioner: intelliflash-csi-block-driver.intelliflash.com
mountOptions:                        # list of options for `mount -o ...` command
#  - noatime                         #
#- matchLabelExpressions:            # use to following lines to configure topology by zones
#  - key: topology.kubernetes.io/zone
#    values:
#    - us-east
```

#### Example

Run Nginx pod with dynamically provisioned volume:

```bash
kubectl apply -f examples/kubernetes/nginx-dynamic-volume.yaml

# to delete this pod:
kubectl delete -f examples/kubernetes/nginx-dynamic-volume.yaml
```

### Pre-provisioned volumes

The driver can use already existing Intelliflash volumes,
in this case, _StorageClass_, _PersistentVolume_ and _PersistentVolumeClaim_ should be configured.

#### _StorageClass_ configuration

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: intelliflash-csi-driver-cs-nginx-persistent
provisioner: intelliflash-csi-driver.intelliflash.com
mountOptions:                        # list of options for `mount -o ...` command
#  - noatime                         #
```

#### _PersistentVolume_ configuration

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: intelliflash-csi-driver-pv-nginx-persistent
  labels:
    name: intelliflash-csi-driver-pv-nginx-persistent
spec:
  storageClassName: intelliflash-csi-driver-cs-nginx-persistent
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 1Gi
  csi:
    driver: intelliflash-csi-driver.intelliflash.com
    volumeHandle: array1:csi-block-test1:nginx-persistent
  #mountOptions:  # list of options for `mount` command
  #  - noatime    #
```

CSI Parameters:

| Name           | Description                                                       | Example                              |
|----------------|-------------------------------------------------------------------|--------------------------------------|
| `driver`       | installed driver name "intelliflash-csi-block-driver.intelliflash.com"        | `intelliflash-csi-block-driver.intelliflash.com` |
| `volumeHandle` | NS appliance name from config and path to existing Intelliflash volume [configName:project:volume] | `array1:csi-block-test1:nginx-persistent`               |

#### _PersistentVolumeClaim_ (pointed to created _PersistentVolume_)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: intelliflash-csi-driver-pvc-nginx-persistent
spec:
  storageClassName: intelliflash-csi-driver-cs-nginx-persistent
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:
      # to create 1-1 relationship for pod - persistent volume use unique labels
      name: intelliflash-csi-block-driver-pv-nginx-persistent
```

#### Example

Run nginx server using PersistentVolume.

**Note:** Pre-configured volume should exist on the Intelliflash:
`csi-block-test1:nginx-persistent`.

```bash
kubectl apply -f examples/kubernetes/nginx-persistent-volume.yaml

# to delete this pod:
kubectl delete -f examples/kubernetes/nginx-persistent-volume.yaml
```

### Cloned volumes

We can create a clone of an existing csi volume.
To do so, we need to create a _PersistentVolumeClaim_ with _dataSource_ spec pointing to an existing PVC that we want to clone.
In this case Kubernetes generates volume name automatically (for example `pvc-ns-cfc67950-fe3c-11e8-a3ca-005056b857f8`).

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: intelliflash-csi-block-driver-pvc-nginx-dynamic-clone
spec:
  storageClassName: intelliflash-csi-block-driver-cs-nginx-dynamic
  dataSource:
    kind: PersistentVolumeClaim
    apiGroup: ""
    name: intelliflash-csi-block-driver-pvc-nginx-dynamic # pvc name
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

#### Example

Run Nginx pod with dynamically provisioned volume:

```bash
kubectl apply -f examples/kubernetes/nginx-clone-volume.yaml

# to delete this pod:
kubectl delete -f examples/kubernetes/nginx-clone-volume.yaml
```

## Snapshots

**Note**: this feature is an
[alpha feature](https://kubernetes-csi.github.io/docs/snapshot-restore-feature.html#status).

```bash
# create snapshot class
kubectl apply -f examples/kubernetes/snapshot-class.yaml

# take a snapshot
kubectl apply -f examples/kubernetes/take-snapshot.yaml

# deploy nginx pod with volume restored from a snapshot
kubectl apply -f examples/kubernetes/nginx-snapshot-volume.yaml

# snapshot classes
kubectl get volumesnapshotclasses.snapshot.storage.k8s.io

# snapshot list
kubectl get volumesnapshots.snapshot.storage.k8s.io

# snapshot content list
kubectl get volumesnapshotcontents.snapshot.storage.k8s.io
```

## CHAP Authentication
- Configure a project (or a separate iSCSI target) with iSCSI CHAP authentication enabled on Intelliflash appliance.
- Create an initiatorGroup on IF array that will be used for CSI initiators.
- Use values from the iSCSI target and initiator group in CSI config yaml. 
Example:
```
  useChapAuth: true                           # Defines whether CHAP authentication is enabled
  chapUser: admin                             # CHAP username
  chapSecret: chapsecretif                    # CHAP secret
  initiatorGroup: csi-chap-initiator-group    # Initiator group associated with project, required for CHAP
```

## InheritProjectSettings option

This option tells the driver whether it should inherit iSCSI options from the project or use what is provided via storage class parameters or config. If this option is set to `true`, no additional settings are required. If you want to use a separate iSCSI target/target group, use `false` and provide the following parameters. For storageClass parameters:
- target
- targetGroup
- iSCSIPort (optional)

Example storageClass:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: intelliflash-csi-block-driver-pvc-nginx-dynamic-clone
spec:
  storageClassName: intelliflash-csi-block-driver-cs-nginx-dynamic
  dataSource:
    kind: PersistentVolumeClaim
    apiGroup: ""
    name: intelliflash-csi-block-driver-pvc-nginx-dynamic # pvc name
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
parameters:
  targetGroup: non-default
  target: non-default
  inheritProjectSettings: "false"
```


For config values a prefix 'Default' is added to each parameter, as follows:
- DefaultTarget
- DefaultTargetGroup
- DefaultISCSIPort

Example config file:
```yaml
arrays:
  array1:
    restIp: https://172.27.10.30:443   # [required] IntelliFlash REST API endpoint
    username: admin                    # [required] IntelliFlash REST API username
    password: t                        # [required] IntelliFlash REST API password
    defaultProject: csi-block          # default project name for driver's volume
    defaultDataIp: 172.27.10.30        # default IntelliFlash data IP
    inheritProjectSettings: false      # defines whether the volume should inherit iscsi mapping from the project or create depending on csi config
    defaultTarget: csi                 # default target to use when inheritProjectSettings=false
    defaultTargetGroup: csi            # default target group to use when inheritProjectSettings=false
    defaultHostGroup: csi              # default host group to use when inheritProjectSettings=false
    defaultISCSIPort: 3260             # default iSCSI port to use when inheritProjectSettings=false

useChapAuth: true                           # Defines whether CHAP authentication is enabled
chapUser: admin                             # CHAP username
chapSecret: chapsecretif                    # CHAP secret
initiatorGroup: csi-block-chap-initiator-group  # Initiator group associated with project, required for CHAP
debug: true
```

## Uninstall

Using the same files as for installation:

```bash
# delete driver
kubectl delete -f deploy/kubernetes/intelliflash-csi-block-driver.yaml

# delete secret
kubectl delete secret intelliflash-csi-block-driver-config
```

## Troubleshooting

- Show installed drivers:
  ```bash
  kubectl get csidrivers
  kubectl describe csidrivers
  ```
- Error:
  ```
  MountVolume.MountDevice failed for volume "pvc-ns-<...>" :
  driver name intelliflash-csi-block-driver.intelliflash.com not found in the list of registered CSI drivers
  ```
  Make sure _kubelet_ configured with `--root-dir=/var/lib/kubelet`, otherwise update paths in the driver yaml file
  ([all requirements](https://github.com/kubernetes-csi/docs/blob/387dce893e59c1fcf3f4192cbea254440b6f0f07/book/src/Setup.md#enabling-features)).
- "VolumeSnapshotDataSource" feature gate is disabled:
  ```bash
  vim /var/lib/kubelet/config.yaml
  # ```
  # featureGates:
  #   VolumeSnapshotDataSource: true
  # ```
  vim /etc/kubernetes/manifests/kube-apiserver.yaml
  # ```
  #     - --feature-gates=VolumeSnapshotDataSource=true
  # ```
  ```
- Driver logs
  ```bash
  kubectl logs -f intelliflash-csi-controller-0 driver
  kubectl logs -f $(kubectl get pods | awk '/intelliflash-csi-node/ {print $1;exit}') driver
  ```
- Show termination message in case driver failed to run:
  ```bash
  kubectl get pod intelliflash-csi-controller-0 -o go-template="{{range .status.containerStatuses}}{{.lastState.terminated.message}}{{end}}"
  ```
- Configure Docker to trust insecure registries:
  ```bash
  # add `{"insecure-registries":["10.3.199.92:5000"]}` to:
  vim /etc/docker/daemon.json
  service docker restart
  ```

## Development

Commits should follow [Conventional Commits Spec](https://conventionalcommits.org).
Commit messages which include `feat:` and `fix:` prefixes will be included in CHANGELOG automatically.

### Build

In order to use private bitbucket packages (go-intelliflash in our case), we need to set git to use ssh instead of https.
```bash
git config --global --add url."ssh://git@bitbucket.eng-us.tegile.com:7999/".insteadOf "https://bitbucket.eng-us.tegile.com/scm/"
```

And set the GOPRIVATE env variable
```bash
export GOPRIVATE=bitbucket.eng-us.tegile.com/eco/go-intelliflash
```

```bash
# print variables and help
make

# build go app on local machine
make build

# build container (+ using build container)
make container-build

# update deps
~/go/bin/dep ensure
```

### Run

Without installation to k8s cluster only version command works:

```bash
./bin/nexentastor-csi-driver --version
```

### Publish

```bash
# push the latest built container to the local registry (see `Makefile`)
make container-push-local

# push the latest built container to hub.docker.com
make container-push-remote
```

