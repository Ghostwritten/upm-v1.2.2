# Installing UPM on Openshift

## Installing UPM on Openshift

Version information

| Version | Date       | Updater      | Notes       |
| ------- | ---------- | ------------ | ----------- |
| 0.1     | 2024-12-23 | Zhang Yongqi | First Draft |
| 0.2     | 2024-12-25 | Zhang Yongqi | Revision    |
| 0.4     | 2024-12-27 | Zhang Yongqi | Revision    |
| 0.5     | 2025-02-07 | Zong Xun     | Revision    |

## 1. Introduction

UPM (Unify Platform Management) is an industry-leading control plane software that supports installation in Kubernetes or Openshift. UPM can manage a variety of types of application software, including RDBMS (MySQL), cache (Redis), non-relational database (ElasticSearch), MQ (Kafka) and middleware (ZooKeeper). At present, the team is actively integrating more types of application software and developing more functions to meet customer customization needs into UPM. It has currently supported the management of 6 application software.

## 2. Environmental and resource requirements

### 2.1 Hardware resource requirements

OCP resource requirements (Control Plane)

|   | Role                                            | Node Type       | Node | Number of CPUs per node (core) | Memory capacity per node (GB) | System disk capacity GB) | Persistent storage PVC capacity (GB) | Operating system                                                    | Virtualization platform support | Note                                                                                                                                |
| - | ----------------------------------------------- | --------------- | ---- | ------------------------------ | ----------------------------- | ------------------------ | ------------------------------------ | ------------------------------------------------------------------- | ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| 1 | basestation (help node), dns, haproxy, registry | Virtual Machine | 1    | 4                              | 16                            | 100                      | 500                                  | RHEL 8.6 or 9.4 x86\_64                                             | VMware ESX 7.0 VMware ESX 8.0   | Secondary node, DNS node, load balancing node , mirror warehouse nodes, deployment nodes, and Minio nodes. The disk must be an SSD. |
| 2 | OCP control plane                               | Virtual machine | 3    | 4                              | 16                            | 100                      | 0                                    | RHCOS (no manual installation required, OCP automatic installation) | VMware ESX 7.0 VMware ESX 8.0   | The control platform configures three nodes to meet high availability requirements. The disk must be an SSD.                        |
|   |                                                 | total           | 4    | 16                             | 64                            | 400                      | 500                                  |                                                                     |                                 |                                                                                                                                     |

DBScale resource requirements (Workers)

|   | Role           | Node Type                | Node | Number of CPUs per node (core) | Memory capacity per node (GB) | System disk capacity GB) | Persistent storage PVC capacity (GB) | Operating system                                                       | Virtualization platform support | Note                                                                                                                                                            |
| - | -------------- | ------------------------ | ---- | ------------------------------ | ----------------------------- | ------------------------ | ------------------------------------ | ---------------------------------------------------------------------- | ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 3 | Control plane  | Physical/Virtual machine | 3    | 16                             | 32                            | 100                      | 1000                                 | RHCOS (no manual installation required, automatic installation of OCP) | VMware ESX 7.0 VMware ESX 8.0   | Control platform configuration three nodes is for high availability Require. The disk must be an SSD.                                                           |
| 4 | Workload plane | Physical/VM              | 3    | 16                             | 64                            | 100                      | 1000                                 | RHCOS (no manual installation required, automatic installation of OCP) | VMware ESX 7.0 VMware ESX 8.0   | According to this configuration, it is expected to support the creation of 10 MySQL sets High Availability Cluster (1 master 2 slave). The disk must be an SSD. |
|   |                | total                    | 6    | 96                             | 288                           | 600                      | 6000                                 |                                                                        |                                 |                                                                                                                                                                 |

Total resource requirements

| Type    | Number of nodes | Total number of CPU Cores | Total memory GB | Total disk GB |
| ------- | --------------- | ------------------------- | --------------- | ------------- |
| OCP     | 4               | 16                        | 64              | 900           |
| DBScale | 6               | 96                        | 288             | 6600          |
| Total   | 10              | 112                       | 352             | 7500          |

### 2.2 Software Requirements

| Name                | Description                                                                                      |
| ------------------- | ------------------------------------------------------------------------------------------------ |
| Kernel              | Version 4.18.x                                                                                   |
| Operating system    | Enterprise linux kernel support 、Red Hat Enterprise Linux 8.9 or above Oracle Linux 8.9 or above |
| Openshift           | 4.14.x , 4.16.x                                                                                  |
| CSI                 | Comply with database data storage requirements: OpenEBS                                          |
| Access network type | NodePort / LoadBalancer                                                                          |
| Backup Storage      | Type: S3 object storage, such as AWS S3 or Minio S3                                              |

### 2.3 Limitations of use

| Support to manage total workload nodes | Max 100  |
| -------------------------------------- | -------- |
| Number of containers running per node  | Max 15   |
| Support management of total containers | Max 1500 |

## 3. Management machine configuration

The functions of the management machine include:

* Storage installation media;
* Local mirror repository or push mirrors to other local mirror repositories;
* Deploy UPM;
* Post-maintenance and management.

Bastion machine configuration:

* Set the cluster environment variable KUBECONFIG, please set it according to the actual installation situation, copy the kubeconfig file of the cluster environment to the bastion machine, the Kubeconfig file path defaults to /root/.kube/config, and the Kubeconfig file permissions are 600.
* The fortress machine installs the corresponding version of oc, kubectl, helm, podman, skopeo and other necessary tools.

## 4. Obtain UPM media

***

How to obtain UPM media in offline environments:

* ✅ Consult an engineer;

Media Directory:

```sql
  .
├── Docs
├── upm-release.v1.2.2
│   ├── LICENSE
│   ├── scripts
│   ├── upm-addons-images
│   ├── upm-deploy-images
│   └── upm-packages-images
├── upm-release.v1.2.2.tar.gz
└── upm-release.v1.2.2.tar.gz.sha512sum
```

Detailed description:

1. **`Docs`** Contains documentation related to UPM products, including but not limited to:
   * Installation Manual
   * Version Description
   * Document resources such as user guides for users to understand and operate products.
2. **`upm-release.v1.2.2`** `upm-release.v1.2.2.tar.gz`The decompressed directory is the installation media for the UPM platform and contains the following key subdirectories and files:
   * **`LICENSE`** The product's use authorization agreement defines the relevant terms and regulations for users to use the software.
   * **`scripts`** Stores installation scripts that contain script files required to automate deployment, configuration, and initialize the platform environment.
   * **`upm-addons-images`** Contains the mirrors required to install additional open source plugins for the UPM platform. For example:
     * Certificate Management
     * Storage plug-in
     * Third-party integration functions, etc.
   * **`upm-deploy-images`** The core media for installing and running the UPM platform, containing the basic image of the platform, used to initialize and deploy the core services of UPM.
   * **`upm-packages-images`** Provides the images required to create database and middleware services. For example:
     * MySQL
     * Redis
     * Kafka
     * Zookeeper
     * Elasticsearch 等。
3. **`upm-release.v1.2.2.tar.gz`** UPM installation media in compressed package format, including the above `upm-release.v1.2.2`All contents of the directory.
4. **`upm-release.v1.2.2.tar.gz.sha512sum`** SHA-512 verification file, used for verification `upm-release.v1.2.2.tar.gz`File integrity ensures that the transfer and download process is not corrupted or tampered with.

## 5. Container mirroring into the library

For offline environments, the required images need to be pushed to a private mirror repository to ensure that the service is running normally. The following content is a private mirror repository `registry.ocp.local:8443`For example.

> Notice
>
> * Private mirror repository support: Currently UPM projects only support using Red Hat Quay mode as private mirror repository.
> * Mirror push process: In an offline environment, after pushing the required mirror to the specified private mirror repository through oc mirror, ensure that the node can access the repository normally.

Through this step, container image resources in offline environments can be used normally to meet project deployment needs.

### 5.1 Upload offline installation files

Upload offline installation files to the basestation node/operatorhub/upm-v1.2.2/ directory for backup.

\[/operatorhub directory must have enough space to store and decompress installation media, or other directories with sufficient free space. 】

### 5.2 Install the skopeo tool

```
$ yum install skopeo -y
```

Output:

```
Installed:
  skopeo-2:1.14.3-0.1.el9.x86_64
Complete!
```

Test login to the local offline mirror repository

Test to log in to the local offline mirror warehouse using username and password

```
$ skopeo login -u init -p 1GqXc9gd5m07yHoAf4OJ6Ihe8xu3SL2r registry.ocp.local:8443
```

Output:

```
Login Succeeded!
```

【Test login successfully. 】

Test using authfile to log in to the local offline mirror repository

```
$ skopeo login --authfile /root/.docker/config.json registry.ocp.local:8443
```

Output:

```
WARN[0000] saving credentials to ~/.docker/config.json, but not using Docker-compatible file format 
Authenticating with existing credentials for registry.ocp.local:8443
Existing credentials are valid. Already logged in to registry.ocp.local:8443
```

【Test login successfully】

### 5.3 Upload offline images

#### 5.3.1 Decompress offline mirror

```
$ export UPM_PKGS="/operatorhub/upm-v1.2.2"
$ cd ${UPM_PKGS}
```

【Directory File】

```
upm-release.v1.2.2.tar.gz #安装文件压缩包
upm-release.v1.2.2.tar.gz.sha512sum #校验文件
```

View the check value

```
$ cat upm-release.v1.2.2.tar.gz.sha512sum 
```

Output:

```
9da1d5efed857a3b75b682ee577c1504919e222ba4aa34630d2ee0f40b61442fd8831d420fd67ad6c12b9ed19778d996c003538bc4d6c591ecc3c7b476d809e6  upm-release.v1.2.2.tar.gz
```

Verify files

```
$ sha512sum upm-release.v1.2.2.tar.gz
```

Output:

```
9da1d5efed857a3b75b682ee577c1504919e222ba4aa34630d2ee0f40b61442fd8831d420fd67ad6c12b9ed19778d996c003538bc4d6c591ecc3c7b476d809e6  upm-release.v1.2.2.tar.gz
```

\[Confirm that the check value is the same as the check value file above. 】

Unzip the file

```
$ tar zxvf upm-release_v1.2.2.tar.gz
```

【Directory File】

```
upm-release.v1.2.2 #解压后的安装目录
upm-release.v1.2.2.tar.gz
upm-release.v1.2.2.tar.gz.sha512sum
```

```
$ cd  ${UPM_PKGS}/upm-release.v1.2.2
```

【Directory File】

```
LICENSE
scripts
upm-addons-images
upm-deploy-images
upm-packages-images
```

#### 5.3.2 Upload addons image

```
$ export UPLOAD_REGISTRY="registry.ocp.local:8443"
$ export UPM_HOME="/operatorhub/upm-v1.2.2/upm-release.v1.2.2"
$ cd  ${UPM_HOME}/upm-addons-images
```

【Directory File】

```
images
IMAGES_LIST
upload_images.sh  #镜像上传脚本
```

```
$ sh upload_images.sh
```

Output:

```
Getting image list signatures
Copying 18 images generated from 18 images in list
Copying image sha256:79c9716db559ffde1170a4faf04910a08d930f511e6904c4899a1f7be2abfb34 (1/18)
……
……
Copying image sha256:842e5fd1fb60c031f165db5cd26ff6c8b6b1ddae4610775b49d04b654ce64182 (4/4)
Getting image source signatures
Copying blob 83cee2ac931c done   | 
Copying config 3e84f63e1e done   | 
Writing manifest to image destination
Writing manifest list to image destination
Storing list signatures
[Info][2024-12-23T14:52:58+0800]: Upload images done!!!
```

【See Upload images done indicates that the upload is completed. 】

#### 5.3.2 Upload deploy image

```
$ cd ${UPM_HOME}/upm-deploy-images
```

【Directory File】

```
images
IMAGES_LIST
upload_images.sh #镜像上传脚本
yaml
```

```
$ sh upload_images.sh
```

Output:

```
Getting image list signatures
Copying 4 images generated from 4 images in list
Copying image sha256:3057ad3a3b36e8a50178831ee96dbd639b2d37337901f0f14c72ad2db8d80dd7 (1/4)
……
……
[Info][2024-12-23T14:56:20+0800]: Upload images done!!!
[Info][2024-12-23T14:56:20+0800]: Generate upm-imageTagMirrorSet.yaml file done!!!
[Info][2024-12-23T14:56:20+0800]: if install UPM on Openshift. Please apply upm-imageTagMirrorSet.yaml file to your cluster.
[Info][2024-12-23T14:56:20+0800]: Please run cmd: oc apply -f /tmp/upm-imageTagMirrorSet.yaml 
[Info][2024-12-23T14:56:20+0800]: Generate upm-imageDigestMirrorSet.yaml file done!!!
[Info][2024-12-23T14:56:20+0800]: if install UPM on Openshift. Please apply upm-imageDigestMirrorSet.yaml file to your cluster.
[Info][2024-12-23T14:56:20+0800]: Please run cmd: oc apply -f /tmp/upm-imageDigestMirrorSet.yaml 
```

【See Upload images done indicates that the upload is completed. 】

> Note that after the Upload images done is completed, two commands need to be executed manually.

Update imageTagMirrorSet

```
$ cat /tmp/upm-imageTagMirrorSet.yaml 
```

Output:

```
apiVersion: config.openshift.io/v1
kind: ImageTagMirrorSet
metadata:
  annotations:
    description: "Replace UPM mirrors with internal registry"
    environment: "production"
  generation: 1
  name: upm-mirrorset
spec:
  imageTagMirrors:
    - mirrors:
        - registry.ocp.local:8443/upmio
      source: quay.io/upmio
    - mirrors:
        - registry.ocp.local:8443/library
      source: docker.io/library
    - mirrors:
        - registry.ocp.local:8443/rancher
      source: docker.io/rancher
    - mirrors:
        - registry.ocp.local:8443/bitnami
      source: docker.io/bitnami
    - mirrors:
        - registry.ocp.local:8443/nacos
      source: docker.io/nacos
    - mirrors:
        - registry.ocp.local:8443/prometheuscommunity
      source: quay.io/prometheuscommunity
    - mirrors:
        - registry.ocp.local:8443/prometheus
      source: quay.io/prometheus
    - mirrors:
        - registry.ocp.local:8443/panquest
      source: quay.io/panquest
```

```
$ oc apply -f /tmp/upm-imageTagMirrorSet.yaml 
```

Output:

```
imagetagmirrorset.config.openshift.io/upm-mirrorset created
```

Update imageDigestMirrorSet

```
$ cat /tmp/upm-imageDigestMirrorSet.yaml 
```

Output:

```
apiVersion: config.openshift.io/v1
kind: ImageDigestMirrorSet
metadata:
  name: upm-mirrorset
spec:
  imageDigestMirrors:
    - mirrors:
        - registry.ocp.local:8443/jetstack
      source: quay.io/jetstack
    - mirrors:
        - registry.ocp.local:8443/cloudnative-pg
      source: ghcr.io/cloudnative-pg
    - mirrors:
        - registry.ocp.local:8443/upmio
      source: quay.io/upmio
```

```
$ oc apply -f /tmp/upm-imageDigestMirrorSet.yaml 
```

Output:

```
imagedigestmirrorset.config.openshift.io/upm-mirrorset created
```

Check mcp, wait and confirm that the update is completed

```
$ oc get mcp
```

Output:

```
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-d0f3be6db3842b7d33a732d5a74dc0fd   True      False      False      3              3                   3                     0                      2d2h
worker   rendered-worker-5f2b057424077845b7e0bfd2e08a3b9b   True      False      False      6              6                   6                     0                      2d2h
```

\[Seeing the number of UPDATEDMACHINECOUNTs is the same as that of master and worker, it means that the update is completed. 】

```
$ oc describe co/machine-config | grep -A 2 Extension
```

Output:

```
  Extension:
    Master:  all 3 nodes are at latest configuration rendered-master-d0f3be6db3842b7d33a732d5a74dc0fd
    Worker:  all 6 nodes are at latest configuration rendered-worker-5f2b057424077845b7e0bfd2e08a3b9b
```

\[Seeing the information above means that the update is completed. 】

Check the worker node registry and confirm that it has been updated

```
$ ssh core@192.168.35.151
$ cat /etc/containers/registries.conf | grep upm
```

Output:

```
  location = "quay.io/upmio"
    location = "registry.ocp.local:8443/upmio"
    location = "registry.ocp.local:8443/upmio"
```

\[See the upmio information above that the registry update is completed. 】

#### 5.3.2 Upload packages image

```
$ cd ${UPM_HOME}/upm-packages-images
```

【Directory File】

```
images
IMAGES_LIST
upload_images.sh	#镜像上传脚本
```

```
sh upload_images.sh
```

Output:

```
Getting image list signatures
Copying 4 images generated from 4 images in list
Copying image sha256:1c9a893d7190e77356aafc28df5c037a10c1fa9aba939b5e08054b17c786dae0 (1/4)
……
……
Writing manifest list to image destination
Storing list signatures
[Info][2024-12-23T15:27:51+0800]: Upload images done!!!
```

【See Upload images done indicates that the upload is completed. 】

> **Reinstalling UPM starts from this step**

Disable Online OperatorHub

```
$ vim /root/post-ocp.sh
```

Output:

```
oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=/root/.docker/config.json
oc get operatorhubs.config.openshift.io -o yaml
oc patch OperatorHub cluster --type json -p '[{"op": "add", "path": "/spec/disableAllDefaultSources", "value": true}]'
oc get operatorhubs.config.openshift.io -o yaml
```

```
$ sh /root/post-ocp.sh
```

Edited content:

```
info: pull-secret was not changed
……
……
  spec:
    disableAllDefaultSources: true
  status:
    sources:
    - disabled: true
      name: redhat-operators
      status: Success
    - disabled: true
      name: certified-operators
      status: Success
    - disabled: true
      name: community-operators
      status: Success
    - disabled: true
      name: redhat-marketplace
      status: Success
……
……  
```

Update the mirror map

```
$ oc apply -f /tmp/upm-imageTagMirrorSet.yaml 
```

Output:

```
imagetagmirrorset.config.openshift.io/upm-mirrorset configured
```

```
$ oc apply -f /tmp/upm-imageDigestMirrorSet.yaml 
```

Output:

```
imagedigestmirrorset.config.openshift.io/upm-mirrorset configured
```

## 6. Install UPM

\[Before starting to install UPM, you must first complete the storage CSI configuration work in accordance with the "OCP-4.14 Offline Installation of OpenEBS (lvm\_localpv) operator manual". 】

### 6.1 Install upm-platfrom

```
$ export UPM_HOME="/operatorhub/upm-v1.2.2/upm-release.v1.2.2"
```

Set up the upm-platform configuration file, the example content is as follows:

```
$ cd ${UPM_HOME}/scripts/platform
```

【Directory File】

```
local-path-provisioner-install.sh
local-path-provisioner-uninstall.sh
run.sh
upm-install.sh	#upm安装脚本
upm-uninstall.sh
yaml
```

Modify environment variables

```
$ vim  yaml/upm/env.yaml
```

Edited content:

```bash
imageRegistry: "quay.io"
namespace: "upm-system"
upm:
  nodeNames: "worker1"
  mysql:
    password: "UPM@2024!"

nacos:
  storageClassName: "openebs-lvmsc-ssd"
  nodeNames: "worker2"
  mysql:
    password: "UPM@2024!"

mysql:
  storageClassName: "openebs-lvmsc-ssd"
  nodeNames: "worker3"
  rootPassword: "UPM@2024!"

redis:
  nodeNames: "worker3"
  authPassword: "UPM@2024!"
  storageClassName: "openebs-lvmsc-ssd"
```

Parameter description:

Note that storageClassName is storageClass in k8s. Please use the storageClass that the environment actually uses.

* `imageRegister`: Configure the image repository name (do not modify it, because the mirror forwarding configuration has been used);
* `upm.nodeNames`: Deploy the upm-control scheduling to the specified node, which is set to worker1 here;
* `upm.mysql.password`: upm-control connection mysql password;
* `nacos.storageClassName`: Deploy nacos is applied to the specified storage object storageClassName, which should be changed to the same as the deployed openEBS (lvm-localpv) openebs-lvmsc-ssd;

How to get storageClass

```
  $ oc get sc
```

\[NAME column is the modified SC noun. 】

```
  NAME                PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
  openebs-lvmsc-ssd   local.csi.openebs.io   Delete          WaitForFirstConsumer   true                   47h
```

* `nacos.nodeNames`: Deploy the nacos schedule to the specified node, which is set to worker2 here;
* `nacos.mysql.password`: Nacos connection mysql password;
* `mysql.storageClassName`: Deploy MySQL to apply to the specified storage object storageClassName;
* `mysql.nodeNames`: Deploy MySQL scheduling to the specified node, set to worker3 here;
* `mysql.rootPassword`: MySQL root password
* `redis.nodeNames`: Deploy the reids schedule to the specified node, which is set to worker3 here;
* `redis.authPassword`: redis password

Copy kube config and modify permissions

```
$ cp /root/.kube/config . && chmod 644 config 
```

Modify the installation script

```
$ vim upm-install.sh
```

Will

```
KUBECONFIG="${HOME}/.kube/config"
REGISTRY="quay.io"
```

Change to

```
KUBECONFIG="./config"
REGISTRY="registry.ocp.local:8443"
```

Execute the installation script

```
$ sh -x upm-install.sh v1.2.2
```

Output:

```
+ set -o nounset
+ VERSION=v1.2.2
+ NAME=upm-platform-deploy
+ KUBECONFIG=./config
+ REGISTRY=registry.ocp.local:8443
+ IMAGE=upmio/upm-platform-deploy
+ UI_VERSION=v1.2.2
+ API_SERVER_VERSION=v1.2.2
+ command -v docker
+ command -v podman
+ CRI=podman
+ podman run --rm -it --network=host --name upm-platform-deploy -v ./config:/tmp/kubeconfig:ro -v ./yaml:/tmp/yaml --env UI_VERSION=v1.2.2 --env API_SERVER_VERSION=v1.2.2 --env ENV_YAML=/tmp/yaml/upm/env.yaml --env KUBECONFIG=/tmp/kubeconfig registry.ocp.local:8443/upmio/upm-platform-deploy:v1.2.2 ./upm-install.sh
[Info][2024-12-23T16:47:20+0800]: Log file created at /tmp/upm-platform-install-2024-12-23_16-47-20.log
[Info][2024-12-23T16:47:20+0800]: Current Kubernetes context:
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://api.demo.ocp.local:6443
  name: api-demo-ocp-local:6443
contexts:
- context:
    cluster: api-demo-ocp-local:6443
    namespace: openshift-operators
    user: system:admin/api-demo-ocp-local:6443
  name: openshift-operators/api-demo-ocp-local:6443/system:admin
current-context: openshift-operators/api-demo-ocp-local:6443/system:admin
kind: Config
preferences: {}
users:
- name: system:admin/api-demo-ocp-local:6443
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
Are you sure to install upm-platform on this kubernetes cluster? [y/N] y
#这里要输入 y 继续
[Info][2024-12-23T16:48:16+0800]: Creating namespace upm-system...
namespace/upm-system created
[Info][2024-12-23T16:48:16+0800]: Namespace upm-system created
[Info][2024-12-23T16:48:16+0800]: Openshift detected, patching security context constraints...
role.rbac.authorization.k8s.io/scc-role created
[Info][2024-12-23T16:48:16+0800]: Role scc-role created
rolebinding.rbac.authorization.k8s.io/scc-rolebinding created
[Info][2024-12-23T16:48:17+0800]: RoleBinding scc-rolebinding created
Warning: resource serviceaccounts/default is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply. kubectl apply should only be used on resources created declaratively by either kubectl create --save-config or kubectl apply. The missing annotation will be patched automatically.
serviceaccount/default configured
[Info][2024-12-23T16:48:17+0800]: ServiceAccount default created
[Info][2024-12-23T16:48:17+0800]: Install upm-platform, It might take a long time...
……
……
NOTES:
Successfully installed upm-platform.

Check the status by running: kubectl get pods -n upm-system
[Info][2024-12-23T16:53:57+0800]: Wait for upm-platform ready...
Waiting for all pods to be ready or succeeded...
Waiting for all pods to be ready or succeeded...
Waiting for all pods to be ready or succeeded...
Waiting for all pods to be ready or succeeded...
Waiting for all pods to be ready or succeeded...
Waiting for all pods to be ready or succeeded...
Waiting for all pods to be ready or succeeded...
Waiting for all pods to be ready or succeeded...
[Info][2024-12-23T16:56:39+0800]: All pods are ready or succeeded
NAME                                             READY   STATUS      RESTARTS        AGE     IP             NODE      NOMINATED NODE   READINESS GATES
upm-platform-auth-6cf844b675-2s7k8               1/1     Running     0               8m20s   10.86.14.43    worker1   <none>           <none>
upm-platform-cnpg-ms-6d765cd7f-v8hc5             1/1     Running     0               8m20s   10.86.14.44    worker1   <none>           <none>
upm-platform-elasticsearch-ms-77f49b9794-94d2n   1/1     Running     0               8m20s   10.86.14.42    worker1   <none>           <none>
upm-platform-gateway-744c94dcfc-kzd8f            1/1     Running     0               8m20s   10.86.14.51    worker1   <none>           <none>
upm-platform-kafka-ms-84567b5cf-qr77k            1/1     Running     0               8m20s   10.86.14.49    worker1   <none>           <none>
upm-platform-mysql-0                             1/1     Running     0               8m20s   10.86.10.24    worker3   <none>           <none>
upm-platform-mysql-ms-54456bbd59-wl7xl           1/1     Running     0               8m20s   10.86.14.50    worker1   <none>           <none>
upm-platform-nacos-0                             1/1     Running     2 (4m22s ago)   8m20s   10.86.12.223   worker2   <none>           <none>
upm-platform-nacos-init-db-bgrwm                 0/1     Completed   0               8m20s   10.86.12.222   worker2   <none>           <none>
upm-platform-nginx-549bf6844c-v2lq6              1/1     Running     0               8m20s   10.86.14.41    worker1   <none>           <none>
upm-platform-operatelog-7c7579c7f9-545hc         1/1     Running     0               8m20s   10.86.14.54    worker1   <none>           <none>
upm-platform-postgresql-ms-f7dcd5d65-q9lq9       1/1     Running     0               8m20s   10.86.14.47    worker1   <none>           <none>
upm-platform-redis-cluster-ms-554949884f-c2xzv   1/1     Running     0               8m20s   10.86.14.48    worker1   <none>           <none>
upm-platform-redis-master-0                      1/1     Running     0               8m20s   10.86.10.25    worker3   <none>           <none>
upm-platform-redis-ms-d8886f498-z9tvs            1/1     Running     0               8m20s   10.86.14.40    worker1   <none>           <none>
upm-platform-resource-77df98ddb5-bw9nj           1/1     Running     0               8m20s   10.86.14.53    worker1   <none>           <none>
upm-platform-ui-6dbf77fd86-tzvls                 1/1     Running     0               8m20s   10.86.14.46    worker1   <none>           <none>
upm-platform-user-6665cf4b64-w95rb               1/1     Running     0               8m20s   10.86.14.45    worker1   <none>           <none>
upm-platform-zookeeper-ms-558dc7c7f5-j8tpf       1/1     Running     0               8m20s   10.86.14.52    worker1   <none>           <none>
[Info][2024-12-23T16:56:39+0800]: upm-platform ready, elapsed time: 162 seconds
```

【Seeing upm-platform ready means the installation is successful. The installation process is about 5 to 10 minutes. 】

Open a new terminal and track the creation progress

```
$ kubectl get pod -n upm-system -w
```

Output:

```
NAME                                             READY   STATUS                  RESTARTS   AGE
upm-platform-auth-6cf844b675-x6m5f               0/1     Init:ImagePullBackOff   0          9m
upm-platform-cnpg-ms-6d765cd7f-6gxnt             0/1     Init:ImagePullBackOff   0          9m
upm-platform-elasticsearch-ms-77f49b9794-fzjlz   0/1     Init:ImagePullBackOff   0          9m1s
upm-platform-gateway-744c94dcfc-ln2t5            0/1     Init:ImagePullBackOff   0          9m
upm-platform-kafka-ms-84567b5cf-zcx4j            0/1     Init:ImagePullBackOff   0          9m
upm-platform-mysql-0                             0/1     Init:ImagePullBackOff   0          9m
upm-platform-mysql-ms-54456bbd59-98ml6           0/1     Init:ImagePullBackOff   0          9m
upm-platform-nacos-0                             0/1     Init:ImagePullBackOff   0          9m
upm-platform-nacos-init-db-xbpgl                 0/1     ImagePullBackOff        0          9m1s
upm-platform-nginx-549bf6844c-j7mhp              0/1     ImagePullBackOff        0          9m1s
upm-platform-operatelog-7c7579c7f9-n9vfg         0/1     Init:ImagePullBackOff   0          9m
upm-platform-postgresql-ms-f7dcd5d65-r4dvj       0/1     Init:ImagePullBackOff   0          9m1s
upm-platform-redis-cluster-ms-554949884f-fwddk   0/1     Init:ImagePullBackOff   0          9m1s
upm-platform-redis-master-0                      0/1     ImagePullBackOff        0          9m
upm-platform-redis-ms-d8886f498-ghkvf            0/1     Init:ImagePullBackOff   0          9m1s
upm-platform-resource-77df98ddb5-dhvwc           0/1     Init:ImagePullBackOff   0          9m
upm-platform-ui-6dbf77fd86-s46mw                 0/1     ImagePullBackOff        0          9m
upm-platform-user-6665cf4b64-n2ldx               0/1     Init:ImagePullBackOff   0          9m
upm-platform-zookeeper-ms-558dc7c7f5-p6hg9       0/1     Init:ImagePullBackOff   0          9m
```

\[During the installation process, the pod status may be ImagePullBackOff, which will be ignored and will be automatically repaired. 】

examine `upm-system`Pod status of all components.

```
$ kubectl get pod -n upm-system 
```

Output:

```
NAME                                             READY   STATUS      RESTARTS      AGE
upm-platform-auth-6cf844b675-2s7k8               1/1     Running     0             14m
upm-platform-cnpg-ms-6d765cd7f-v8hc5             1/1     Running     0             14m
upm-platform-elasticsearch-ms-77f49b9794-94d2n   1/1     Running     0             14m
upm-platform-gateway-744c94dcfc-kzd8f            1/1     Running     0             14m
upm-platform-kafka-ms-84567b5cf-qr77k            1/1     Running     0             14m
upm-platform-mysql-0                             1/1     Running     0             14m
upm-platform-mysql-ms-54456bbd59-wl7xl           1/1     Running     0             14m
upm-platform-nacos-0                             1/1     Running     2 (10m ago)   14m
upm-platform-nacos-init-db-bgrwm                 0/1     Completed   0             14m
upm-platform-nginx-549bf6844c-v2lq6              1/1     Running     0             14m
upm-platform-operatelog-7c7579c7f9-545hc         1/1     Running     0             14m
upm-platform-postgresql-ms-f7dcd5d65-q9lq9       1/1     Running     0             14m
upm-platform-redis-cluster-ms-554949884f-c2xzv   1/1     Running     0             14m
upm-platform-redis-master-0                      1/1     Running     0             14m
upm-platform-redis-ms-d8886f498-z9tvs            1/1     Running     0             14m
upm-platform-resource-77df98ddb5-bw9nj           1/1     Running     0             14m
upm-platform-ui-6dbf77fd86-tzvls                 1/1     Running     0             14m
upm-platform-user-6665cf4b64-w95rb               1/1     Running     0             14m
upm-platform-zookeeper-ms-558dc7c7f5-j8tpf       1/1     Running     0             14m
```

\[There are 19 pods in total, 18 pods are running, and 1 pod state is Completed. 】

Check PV and PVC

```
$ oc get pv
```

Output:

```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                               STORAGECLASS        REASON   AGE
pvc-837eeada-60df-46d4-b94d-cbe3c3fd0548   8Gi        RWO            Delete           Bound    upm-system/redis-data-upm-platform-redis-master-0   openebs-lvmsc-ssd            81m
pvc-a3ccca6e-18e1-4097-9fe0-dfc6f9196bad   50Gi       RWO            Delete           Bound    upm-system/data-upm-platform-mysql-0                openebs-lvmsc-ssd            81m
pvc-efddf7da-f06f-40cc-9ca1-b6aaaf904b0f   47Gi       RWO            Delete           Bound    upm-system/data-storage-upm-platform-nacos-0        openebs-lvmsc-ssd            81m
```

【STATUS for Bound indicates that the pv is created successfully. 】

```
$ oc get pvc -n upm-system 
```

Output:

```
NAMESPACE    NAME                                     STATUS   VOLUME                                    CAPACITY   ACCESS MODES   STORAGECLASS        AGE
upm-system   data-storage-upm-platform-nacos-0        Bound    pvc-efddf7da-f06f-40cc-9ca1-b6aaaf904b0f   47Gi       RWO            openebs-lvmsc-ssd   81m
upm-system   data-upm-platform-mysql-0                Bound    pvc-a3ccca6e-18e1-4097-9fe0-dfc6f9196bad   50Gi       RWO            openebs-lvmsc-ssd   81m
upm-system   redis-data-upm-platform-redis-master-0   Bound    pvc-837eeada-60df-46d4-b94d-cbe3c3fd0548   8Gi        RWO            openebs-lvmsc-ssd   81m
```

【STATUS is Bound, ACCESS MODES is RWO, indicating that pvc mounts are normal. 】

\[Optional steps: Log in to the OCP console, access "Storage", view "Permanent Volume" and "Permanent Volume Statement". You can also see the situation of PV and PVC. 】

### 6.2 Install upm-engine

set up `upm-engine`The configuration file, the example content is as follows:

```
$ cd ${UPM_HOME}/scripts/engine/
```

【Directory File】

```
cert-manager-install.sh
cert-manager-uninstall.sh
cnpg-install.sh
cnpg-uninstall.sh
run.sh
upm-install.sh	#upm安装脚本
upm-uninstall.sh
yaml
```

Modify environment variables

```
$ vim yaml/upm/env.yaml
```

Output:

```
imageRegistry: "quay.io"
upm-engine:
  nodeNames: "worker2"
```

`upm-engine`Required parameter description for component installation:

* `imageRegistry`: Configure the container image repository name (do not modify it, because the mirror forwarding configuration has been used).
* `upm-engine.nodeNames`: Deploy the upm-engine scheduling to the specified node, which is set to worker2 here.

Copy kube config and modify permissions

```
$ cp /root/.kube/config . && chmod 644 config 
```

Modify the installation script

```
$ vim upm-install.sh
```

Will

```
KUBECONFIG="${HOME}/.kube/config"
REGISTRY="quay.io"
```

Change to

```
KUBECONFIG="./config"
REGISTRY="registry.ocp.local:8443"
```

Execute the installation script

```
$ sh -x upm-install.sh v1.2.2
```

Output:

```
+ set -o nounset
+ VERSION=v1.2.2
+ NAME=upm-engine-deploy
+ KUBECONFIG=./config
+ REGISTRY=registry.ocp.local:8443
+ IMAGE=upmio/upm-engine-deploy
+++ dirname upm-install.sh
++ cd .
++ pwd
+ SCRIPT_DIR=/operatorhub/upm-v1.2.2/upm-release.v1.2.2/scripts/engine
+ TESSERACT_CUBE_VERSION=v1.2.2
+ KAUNTLET_VERSION=v1.2.2
+ TEMPLATE_VERSION=v1.2.2
+ command -v docker
+ command -v podman
+ CRI=podman
+ podman run --rm -it --network=host --name upm-engine-deploy -v ./config:/tmp/kubeconfig:ro -v /operatorhub/upm-v1.2.2/upm-release.v1.2.2/scripts/engine/yaml:/tmp/yaml --env TESSERACT_CUBE_VERSION=v1.2.2 --env KAUNTLET_VERSION=v1.2.2 --env TEMPLATE_VERSION=v1.2.2 --env ENV_YAML=/tmp/yaml/upm/env.yaml --env KUBECONFIG=/tmp/kubeconfig registry.ocp.local:8443/upmio/upm-engine-deploy:v1.2.2 ./upm-install.sh
Trying to pull registry.ocp.local:8443/upmio/upm-engine-deploy:v1.2.2...
[Info][2024-12-23T17:17:01+0800]: Configuring node labels for UPM Engine...
[Info][2024-12-23T17:17:01+0800]: Successfully labeled node worker2 for UPM Engine
[Info][2024-12-23T17:17:01+0800]: Labeled nodes for UPM Engine
[Info][2024-12-23T17:17:01+0800]: Current Kubernetes context:
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://api.demo.ocp.local:6443
  name: api-demo-ocp-local:6443
contexts:
- context:
    cluster: api-demo-ocp-local:6443
    namespace: openshift-operators
    user: system:admin/api-demo-ocp-local:6443
  name: openshift-operators/api-demo-ocp-local:6443/system:admin
current-context: openshift-operators/api-demo-ocp-local:6443/system:admin
kind: Config
preferences: {}
users:
- name: system:admin/api-demo-ocp-local:6443
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
Are you sure to install UPM Engine on this kubernetes cluster? [y/N] y
#这里要输入 y 继续
[Info][2024-12-23T17:20:01+0800]: Installing UPM Engine on OpenShift...
operatorgroup.operators.coreos.com/upm-system-hv815 created
[Info][2024-12-23T17:20:01+0800]: Adding OpenShift operator tesseract-cube version v1.2.2...
catalogsource.operators.coreos.com/tesseract-cube-catalog created
[Info][2024-12-23T17:20:01+0800]: Waiting for tesseract-cube CatalogSource pod to be ready... (0/60)
……
……
subscription.operators.coreos.com/tesseract-cube-operator created
[Info][2024-12-23T17:20:12+0800]: Waiting for tesseract-cube CSV to be ready... (0/60)
……
……
[Info][2024-12-23T17:20:52+0800]: Successfully installed tesseract-cube operator version v1.2.2
[Info][2024-12-23T17:20:52+0800]: Adding OpenShift operator kauntlet version v1.2.2...
catalogsource.operators.coreos.com/kauntlet-catalog created
[Info][2024-12-23T17:20:52+0800]: Waiting for kauntlet CatalogSource pod to be ready... (0/60)
subscription.operators.coreos.com/kauntlet-operator created
……
……
[Info][2024-12-23T17:21:43+0800]: Successfully installed kauntlet operator version v1.2.2
[Info][2024-12-23T17:21:43+0800]: Importing ConfigMaps for OpenShift...
serviceaccount/upm-engine-import-configmaps created
clusterrole.rbac.authorization.k8s.io/upm-engine-import-configmaps created
clusterrolebinding.rbac.authorization.k8s.io/upm-engine-import-configmaps created
No resources found in ${NAMESPACE} namespace.
job.batch/upm-engine-import-configmaps created
[Info][2024-12-23T17:21:43+0800]: Waiting for ConfigMap import job to complete...
……
……
[Info][2024-12-23T17:22:04+0800]: Successfully imported ConfigMaps
[Info][2024-12-23T17:22:04+0800]: OpenShift-specific installation completed
```

\[Seeing the installation completed above indicates that the installation is successful. 】

Check CSV

```
$ oc get csv
```

Output:

```
NAME                    DISPLAY          VERSION   REPLACES   PHASE
kauntlet.v1.2.2         kauntlet         1.2.2                Succeeded
tesseract-cube.v1.2.2   tesseract-cube   1.2.2                Succeeded
```

\[Seeing the PHASE list of the above 2 CSVs as Succeeded indicates that the installation is successful. 】

examine `upm-system`Pod status of all components.

```
$ kubectl get pods -n upm-system
```

Output:

```
NAME                                                 READY   STATUS    RESTARTS        AGE
kauntlet-controller-manager-74cdc5f96f-47r86         1/1     Running     0             2m50s
tesseract-cube-controller-manager-5f68f9bdc7-5nlm6   1/1     Running     0             3m37s
upm-engine-import-configmaps-kxztn                   0/1     Completed   0             2m25s
upm-platform-auth-6cf844b675-gppv4                   1/1     Running     0             11m
upm-platform-cnpg-ms-6d765cd7f-cg7jx                 1/1     Running     0             11m
upm-platform-elasticsearch-ms-77f49b9794-qdgf2       1/1     Running     0             11m
upm-platform-gateway-744c94dcfc-zls88                1/1     Running     0             11m
upm-platform-kafka-ms-84567b5cf-tgkn4                1/1     Running     0             11m
upm-platform-mysql-0                                 1/1     Running     0             11m
upm-platform-mysql-ms-54456bbd59-t8bjb               1/1     Running     0             11m
upm-platform-nacos-0                                 1/1     Running     3 (10m ago)   11m
upm-platform-nacos-init-db-jrrpl                     0/1     Completed   0             11m
upm-platform-nginx-549bf6844c-tj845                  1/1     Running     0             11m
upm-platform-operatelog-7c7579c7f9-kfk8t             1/1     Running     0             11m
upm-platform-postgresql-ms-f7dcd5d65-l4ljx           1/1     Running     0             11m
upm-platform-redis-cluster-ms-554949884f-rvjxj       1/1     Running     0             11m
upm-platform-redis-master-0                          1/1     Running     0             11m
upm-platform-redis-ms-d8886f498-59q9f                1/1     Running     0             11m
upm-platform-resource-77df98ddb5-7t2p7               1/1     Running     0             11m
upm-platform-ui-6dbf77fd86-6zkf5                     1/1     Running     0             11m
upm-platform-user-6665cf4b64-pll2l                   1/1     Running     0             11m
upm-platform-zookeeper-ms-558dc7c7f5-hj5hg           1/1     Running     0             11m
```

\[Seeing all 22 pods above means the installation is successful, and the top 3 are created after installing engine. 】

kubectl get jobs -n upm-system | grep engine

```
NAME                                                 READY   STATUS      RESTARTS      AGE
upm-engine-import-configmaps-nndzb                   0/1     Completed   0             4m32s
```

\[Seeing the above job upm-engine status is Completed, which means it is normal. 】

【Optional steps】

Log in to the OCP console, click "Operators" = "OperatorsHub" or "Installed Operators" to view the two Operators installed above.

> **Tip: PoC does not require 2 subsequent steps for the time being**

#### Pay special attention

\[The following steps can only be performed in an offline OCP cluster and mirror warehouse environment that is completely deployed by us. If the customer provides an OCP environment that has been deployed, it is necessary to carefully check and communicate with the development team before determining whether it can continue to be executed. 】

```
$ export UPM_HOME="/operatorhub/upm-v1.2.2/upm-release.v1.2.2"
$ cd ${UPM_HOME}/scripts/engine/
```

Update mirrorset Tags

```
$ vim upm-imageTagMirrorSet.yaml 
```

Edited content:

```
apiVersion: config.openshift.io/v1
kind: ImageTagMirrorSet
metadata:
  annotations:
    description: "Replace UPM mirrors with internal registry"
    environment: "production"
  generation: 1
  name: upm-mirrorset
spec:
  imageTagMirrors:
    - mirrors:
        - registry.ocp.local:8443/upmio
      source: quay.io/upmio
    - mirrors:
        - registry.ocp.local:8443/library
      source: docker.io/library
    - mirrors:
        - registry.ocp.local:8443/rancher
      source: docker.io/rancher
    - mirrors:
        - registry.ocp.local:8443/bitnami
      source: docker.io/bitnami
    - mirrors:
        - registry.ocp.local:8443/nacos
      source: docker.io/nacos
    - mirrors:
        - registry.ocp.local:8443/prometheuscommunity
      source: quay.io/prometheuscommunity
    - mirrors:
        - registry.ocp.local:8443/prometheus
      source: quay.io/prometheus
    - mirrors:
        - registry.ocp.local:8443/panquest
      source: quay.io/panquest
    - mirrors:
        - registry.ocp.local:8443/jetstack
      source: quay.io/jetstack
```

```
$ oc apply -f upm-imageTagMirrorSet.yaml 
```

Output:

```
imagetagmirrorset.config.openshift.io/upm-mirrorset configured
```

Update mirrorset Digest

```
$ vim upm-imageDigestMirrorSet.yaml
```

Edited content:

```
apiVersion: config.openshift.io/v1
kind: ImageDigestMirrorSet
metadata:
  name: upm-mirrorset
spec:
  imageDigestMirrors:
    - mirrors:
        - registry.ocp.local:8443/jetstack
      source: quay.io/jetstack
    - mirrors:
        - registry.ocp.local:8443/cloudnative-pg
      source: ghcr.io/cloudnative-pg
    - mirrors:
        - registry.ocp.local:8443/upmio
      source: quay.io/upmio
    - mirrors:
        - registry.ocp.local:8443/enterprisedb
      source: registry.connect.redhat.com/enterprisedb
    - mirrors:
        - registry.ocp.local:8443/community-operator-pipeline-prod
      source: quay.io/community-operator-pipeline-prod
```

```
$ oc apply -f upm-imageDigestMirrorSet.yaml 
```

Output:

```
imagedigestmirrorset.config.openshift.io/upm-mirrorset configured
```

### 6.3 Install cert-manager

#### Pay special attention

If the customer has deployed the cert-manager, do not repeat this step! ! ! 】

set up `cert-manager`The configuration file, the example content is as follows:

```
$ cd ${UPM_HOME}/scripts/engine/
```

Update catalog

```
$ vim catalogSource-cs-community-operator-index.yaml 
```

Output:

```
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: community-operators
  namespace: openshift-marketplace
spec:
  image: registry.ocp.local:8443/redhat/community-operator-index:v4.14
  sourceType: grpc
  displayName: Community Operators
```

【metadata: name must be community-operators】

```
$ oc apply -f catalogSource-cs-community-operator-index.yaml 
```

Output:

```
Warning: resource catalogsources/community-operators is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by oc apply. oc apply should only be used on resources created declaratively by either oc create --save-config or oc apply. The missing annotation will be patched automatically.

catalogsource.operators.coreos.com/community-operators configured
```

Check catalog

```
$ oc get catalogsources.operators.coreos.com -A
```

Output:

```
NAMESPACE               NAME                  DISPLAY               TYPE   PUBLISHER   AGE
openshift-marketplace   community-operators   Community Operators   grpc               17h
……
……
```

【Confirm that the NAME column has the community-operators catalog. 】

Modify environment variables

```
$ vim yaml/cert-manager/env.yaml
```

Output:

```
imageRegistry: "quay.io"
cert-manager:
  nodeNames: "worker3"
```

`cert-manager`Required parameter description for component installation:

* `imageRegistry`: Configure the container image repository name (do not modify it, because the mirror forwarding configuration has been used).
* `cert-manager.nodeNames`: Deploy the cert-manager schedule to the specified node, which is set to worker3 here.

Modify the installation script

```
$ vim cert-manager-install.sh
```

Will

```
KUBECONFIG="${HOME}/.kube/config"
REGISTRY="quay.io"
```

Change to

```
KUBECONFIG="./config"
REGISTRY="registry.ocp.local:8443"
```

Execute the installation script

```
$ sh -x cert-manager-install.sh  v1.2.2
```

Output:

```
+ set -o nounset
+ VERSION=v1.2.2
+ NAME=cert-manager-deploy
+ KUBECONFIG=./config
+ REGISTRY=registry.ocp.local:8443
+ IMAGE=upmio/upm-engine-deploy
+++ dirname cert-manager-install.sh
++ cd .
++ pwd
+ SCRIPT_DIR=/operatorhub/upm-v1.2.2/upm-release.v1.2.2/scripts/engine
+ command -v docker
+ command -v podman
+ CRI=podman
+ podman run --rm -it --network=host --name cert-manager-deploy -v ./config:/tmp/kubeconfig:ro -v /operatorhub/upm-v1.2.2/upm-release.v1.2.2/scripts/engine/yaml:/tmp/yaml --env ENV_YAML=/tmp/yaml/cert-manager/env.yaml --env KUBECONFIG=/tmp/kubeconfig registry.ocp.local:8443/upmio/upm-engine-deploy:v1.2.2 ./cert-manager-install.sh
[Info][2024-12-23T17:33:51+0800]: Log file created at /tmp/cert-manager-install-2024-12-23_17-33-51.log
[Info][2024-12-23T17:33:51+0800]: Current Kubernetes context:
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://api.demo.ocp.local:6443
  name: api-demo-ocp-local:6443
contexts:
- context:
    cluster: api-demo-ocp-local:6443
    namespace: openshift-operators
    user: system:admin/api-demo-ocp-local:6443
  name: openshift-operators/api-demo-ocp-local:6443/system:admin
current-context: openshift-operators/api-demo-ocp-local:6443/system:admin
kind: Config
preferences: {}
users:
- name: system:admin/api-demo-ocp-local:6443
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
Are you sure to install cert-manager on this kubernetes cluster? [y/N] y
#这里要输入 y 继续
[Info][2024-12-23T17:33:54+0800]: Installing cert-manager on OpenShift...
[Info][2024-12-23T17:33:54+0800]: Adding OpenShift operator cert-manager version v1.16.1...
No resources found in openshift-marketplace namespace.
namespace/cert-manager created
operatorgroup.operators.coreos.com/cert-manager-96bhk created
subscription.operators.coreos.com/cert-manager created
[Info][2024-12-23T17:33:55+0800]: Waiting for cert-manager CSV to be ready... (0/30)
[Info][2024-12-24T17:25:25+0800]: Waiting for cert-manager CSV to be ready... (1/30)
[Info][2024-12-24T17:25:36+0800]: Waiting for cert-manager CSV to be ready... (2/30)
[Info][2024-12-24T17:25:47+0800]: Successfully installed cert-manager operator version v1.16.1
```

Check CSV

```
$ oc get csv
```

Output:

```
NAME                     DISPLAY          VERSION   REPLACES                 PHASE
cert-manager.v1.16.1     cert-manager     1.16.1    cert-manager.v1.15.2     Succeeded	#安装成功
……
……
```

\[See cert-manager.v1.16.1 and PHASE is Succeeded that the installation is successful. 】

examine `cert-manager`Pod status of all components.

```
$ kubectl get pod -n cert-manager 
```

Output:

```
cert-manager-cainjector-774b684f98-gsrqd   1/1     Running   0          3m59s
cert-manager-controller-8457454ffc-trrdm   1/1     Running   0          3m59s
cert-manager-webhook-5bfb6df4bf-9nxwn      1/1     Running   0          3m59s
```

Check openshift-marketplace pods

```
$ oc get pods -n openshift-marketplace 
```

Output:

```
NAME                                                              READY   STATUS      RESTARTS   AGE
ad5ca0778fddaee41d26243e9c3d85d96e3882d19499101fbc445aaf189s4lr   0/1     Completed   0          4m
certified-operators-mtqrh                                         1/1     Running     0          16h
community-operators-rf4x4                                         1/1     Running     0          4m23s
marketplace-operator-687946957c-hs4n5                             1/1     Running     0          95s
```

【ad5ca0778 pod status is Completed. 】

Check job

```
$ oc get jobs -n openshift-marketplace 
```

Output:

```
NAME                                                              COMPLETIONS   DURATION   AGE
ad5ca0778fddaee41d26243e9c3d85d96e3882d19499101fbc445aaf18b3213   1/1           4m2s       7m42s
……
……
```

【ad5ca0778 job COMPLETIONS Status 1/1. 】

### 6.4 Install cnpg-operator

```
$ cd ${UPM_HOME}/scripts/engine/
```

Update catalog

```
$ vim catalogSource-cs-certified-operator-index.yaml 
```

Edited content:

```
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: certified-operators
  namespace: openshift-marketplace
spec:
  image: registry.ocp.local:8443/redhat/certified-operator-index:v4.14
  sourceType: grpc
  displayName: certified operators
```

【metadata: name must be certified-operators】

```
$ oc apply -f catalogSource-cs-certified-operator-index.yaml 
```

Output:

```
catalogsource.operators.coreos.com/certified-operators created
```

Check catalog

```
$ oc get catalogsources.operators.coreos.com -A
```

Output:

```
NAMESPACE               NAME                  DISPLAY               TYPE   PUBLISHER   AGE
openshift-marketplace   certified-operators   certified operators   grpc               17h
……
……
```

\[You can see the certified-operators catalog. 】

Modify the installation script

```
$ vim cnpg-install.sh
```

Will

```
KUBECONFIG="${HOME}/.kube/config"
REGISTRY="quay.io"
```

Change to

```
KUBECONFIG="./config"
REGISTRY="registry.ocp.local:8443"
```

Execute the installation script

```
$ sh cnpg-install.sh v1.2.2
```

Output:

```
[Info][2024-12-23T17:47:59+0800]: Log file created at /tmp/cnpg-install-2024-12-23_17-47-59.log
[Info][2024-12-23T17:47:59+0800]: Current Kubernetes context:
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://api.demo.ocp.local:6443
  name: api-demo-ocp-local:6443
contexts:
- context:
    cluster: api-demo-ocp-local:6443
    namespace: openshift-operators
    user: system:admin/api-demo-ocp-local:6443
  name: openshift-operators/api-demo-ocp-local:6443/system:admin
current-context: openshift-operators/api-demo-ocp-local:6443/system:admin
kind: Config
preferences: {}
users:
- name: system:admin/api-demo-ocp-local:6443
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
Are you sure to install CNPG operator on this kubernetes cluster? [y/N] y
#这里要输入 y 继续
[Info][2024-12-23T17:48:02+0800]: Installing cloudnative-pg on OpenShift...
[Info][2024-12-23T17:48:02+0800]: Adding OpenShift operator cnpg version v1.24.1...
No resources found in openshift-marketplace namespace.
namespace/cnpg-system created
operatorgroup.operators.coreos.com/cnpg-system-hv72s created
[Info][2024-12-23T17:48:02+0800]: Configuring cloudnative-pg controller manager...
configmap/cnpg-controller-manager-config created
subscription.operators.coreos.com/cloudnative-pg created
[Info][2024-12-23T17:48:02+0800]: Waiting for cloudnative-pg CSV to be ready... (0/30)
[Info][2024-12-24T16:43:40+0800]: Waiting for cloudnative-pg CSV to be ready... (1/30)
[Info][2024-12-24T16:43:50+0800]: Successfully installed cloudnative-pg operator version v1.24.1
```

【Seeing Successfully installed indicates that the installation is successful. 】

Check csv.

```
$ oc get csv
```

Output:

```
NAME                     DISPLAY          VERSION   REPLACES                 PHASE
cert-manager.v1.16.1     cert-manager     1.16.1    cert-manager.v1.15.2     Succeeded
cloudnative-pg.v1.24.1   CloudNativePG    1.24.1    cloudnative-pg.v1.24.0   Succeeded  #安装成功
kauntlet.v1.2.2          kauntlet         1.2.2                              Succeeded
tesseract-cube.v1.2.2    tesseract-cube   1.2.2                              Succeeded
```

Check openshift-marketplace pods.

```
$ oc get pods -n openshift-marketplace 
```

Output:

```
NAME                                                              READY   STATUS      RESTARTS   AGE
8a72128bcbfca125ed27e3ae1a02ef7f09567558c47e0d36b962d40c8b8d5h6   0/1     Completed   0          9m27s
ad5ca0778fddaee41d26243e9c3d85d96e3882d19499101fbc445aaf18n82gm   0/1     Completed   0          13m
certified-operators-dx5jv                                         1/1     Running     0          10m
community-operators-khdzd                                         1/1     Running     0          13m
kauntlet-catalog-d9vdc                                            1/1     Running     0          17m
marketplace-operator-687946957c-hrx9g                             1/1     Running     0          18m
tesseract-cube-catalog-zqjxv                                      1/1     Running     0          13m
```

\[8a72128bc pod status is Completed. 】

examine `cnpg-operator`Pod status of all components

```
$ kubectl get pod -n cnpg-system 
```

Output:

```
NAME                                       READY   STATUS    RESTARTS   AGE
cnpg-controller-manager-7fd67898bb-dht52   1/1     Running   0          40s
```

【There must be a cnpg-controller-manager pod and it is in the Running state. 】

## 7. Access the interface

* UPM UI format: http://\[master\_ip]:\[port]/upm-ui/#/login

Example:

* http://192.168.35.161:32010/upm-ui/#/login
* Default user: super\_root
* Default password: Upm@2024!

【This manual is finished. 】
