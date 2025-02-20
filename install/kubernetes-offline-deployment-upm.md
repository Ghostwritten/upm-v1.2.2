# Kubernetes Offline Deployment UPM

## Kubernetes Offline Deployment UPM Manual

Version information

| Version | Date       | Updater | Notes    |
| ------- | ---------- | ------- | -------- |
| v1.0    | 2025-02-07 | Zongxun | Revision |

## 1. Introduction

UPM (Unify Platform Management) is an industry-leading control plane software that supports installation in Kubernetes or Openshift. UPM can manage a variety of types of application software, including RDBMS (MySQL), cache (Redis), non-relational database (ElasticSearch), MQ (Kafka) and middleware (ZooKeeper). At present, the team is actively integrating more types of application software and developing more functions to meet customer customization needs into UPM. It has currently supported the management of 6 application software.

## 2. Environmental requirements



Resource Requirements：

| Role            | Node Type                | Node Count | CPU per Node (cores) | Memory per Node (GB) | Persistent Storage PVC Capacity (GB) | Virtualization Platform Support         | Notes                                                                                                                                      |
| --------------- | ------------------------ | ---------- | -------------------- | -------------------- | ------------------------------------ | --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Control Plane   | Physical/Virtual Machine | 3          | 16                   | 32                   | 500                                  | <p>VMware ESX 7.0<br>VMware ESX 8.0</p> | The three-node control plane is configured to meet high availability requirements.                                                         |
| Total           |                          | 3          | 48                   | 96                   | 1500                                 |                                         |                                                                                                                                            |
| Workload Plane  | Physical/Virtual Machine | 3          | 16                   | 64                   | 1000                                 | <p>VMware ESX 7.0<br>VMware ESX 8.0</p> | <p>Based on this configuration, it is expected to support the creation of<br>10 MySQL high-availability clusters (1 master, 2 slaves).</p> |
| Total           |                          | 3          | 48                   | 192                  | 3000                                 |                                         |                                                                                                                                            |
| Deployment Node | Physical/Virtual Machine | 1          | 4                    | 8                    | 500                                  | <p>VMware ESX 7.0<br>VMware ESX 8.0</p> | Can be reclaimed after deployment completion                                                                                               |
| Total           |                          | 1          | 4                    | 8                    | 500                                  |                                         |                                                                                                                                            |



| Software Requirements |                                                                               |
| --------------------- | ----------------------------------------------------------------------------- |
| OS kernel             | version 4.18.x enterprise linux kernel support                                |
| Kubernetes            | v1.27x or above                                                               |
| Openshift             | 4.14.x , 4.16.x                                                               |
| CNI                   | Calico: 3.19.4 or above                                                       |
| Container Runtime     | Containerd 1.7.x or above                                                     |
| CSI                   | Comply with database data storage requirements such as: NFSv3 / ODF / OpenEBS |
| Access network type   | NodePort / LoadBalancer                                                       |
| Backup Storage        | Type: S3                                                                      |

| Limitation of use                      |          |
| -------------------------------------- | -------- |
| Support to manage total workload nodes | Max 100  |
| Number of containers running per node  | Max 15   |
| Support management of total containers | Max 1500 |

## 3. Fortress machine configuration

* The functions of the management machine include:
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
3. **`upm-release.v1.2.2.tar.gz`** UPM installation media in compressed package format, including the above`upm-release.v1.2.2`All contents of the directory.
4. **`upm-release.v1.2.2.tar.gz.sha512sum`** SHA-512 verification file, used for verification`upm-release.v1.2.2.tar.gz`File integrity ensures that the transfer and download process is not corrupted or tampered with.

## 5. Container mirroring into the library

For online environments, this step is not required, and container images can be pulled directly from the Internet's public image repository.

For offline environments, the required images need to be pushed to a private mirror repository to ensure that the service is running normally. The following content is a private mirror repository`registry.upm:35000`For example.

> Notice
>
> * Private mirror repository support: UPM projects are only supported in kubernetes clusters`Docker Registry`As a private mirror repository.
> * Mirror push process: In an offline environment, after pushing the required image to the specified private image repository through Docker, ensure that the node can access the repository normally.

Through this step, container image resources in offline environments can be used normally to meet project deployment needs.

```
$ tar zxvf upm-release_v1.2.2.tar.gz
$ export UPLOAD_REGISTRY="registry.upm:35000"
$ cd upm-release.v1.2.2/upm-addons-images
$ sh upload_images.sh
$ cd upm-release.v1.2.2/upm-deploy-images
$ sh upload_images.sh
$ cd upm-release.v1.2.2/upm-packages-images
$ sh upload_images.sh
```

## 6. Install UPM

### 6.1 Install upm-platfrom

Set up the upm-platform configuration file, the example content is as follows:

```bash
$ vi upm-release_v1.2.2/scripts/platform/yaml/upm/env.yaml
imageRegistry: "quay.io"
namespace: "upm-system"
upm:
  nodeNames: "upm-platform01"
  mysql:
    password: "UPM@2024!"

nacos:
  storageClassName: "local-path"
  nodeNames: "upm-platform01"
  mysql:
    password: "UPM@2024!"

mysql:
  storageClassName: "local-path"
  nodeNames: "upm-platform01"
  rootPassword: "UPM@2024!"

redis:
  nodeNames: "upm-platform01"
  authPassword: "UPM@2024!"
  storageClassName: "local-path"
```

Parameter description:

Note that storageClassName is storageClass in k8s. Please use the storageClass that the environment actually uses.

* `imageRegister`: Configure the image repository name;
* `upm.nodeNames`: Deploy the upm-control scheduling to the specified node;
* `upm.mysql.password`: upm-control connection mysql password;
* `nacos.storageClassName`: Deploy nacos to apply to the specified storage object storageClassName;
* `nacos.nodeNames`: Deploy the nacos schedule to the specified node;
* `nacos.mysql.password`: Nacos connection mysql password;
* `mysql.storageClassName`: Deploy MySQL to apply to the specified storage object storageClassName;
* `mysql.nodeNames`: Deploy MySQL scheduling to the specified node;
* `mysql.rootPassword`: MySQL root password
* `redis.nodeNames`: Deploy the reids schedule to the specified node;
* `redis.authPassword`: redis password

Perform the installation:

```bash
$ cd upm-release_v1.2.2/scripts/platform
$ sh upm-install.sh v1.2.2
```

Open a new terminal and track the creation progress:

```
$ kubectl get pod -n upm-system -w
```

Finally, check`upm-system`Pod status of all components.

```
 NAME                                            READY   STATUS      RESTARTS      AGE
upm-control-auth-6554fc5998-m6259               1/1     Running     1 (39s ago)   9m10s
upm-control-cnpg-ms-684967d956-wltt2            1/1     Running     0             9m9s
upm-control-elasticsearch-ms-5fbf97dc79-5blt2   1/1     Running     0             9m10s
upm-control-gateway-57bbc79b58-9whp6            1/1     Running     0             9m10s
upm-control-kafka-ms-5f5688cccb-6pt6q           1/1     Running     0             9m9s
upm-control-mysql-0                             1/1     Running     0             9m10s
upm-control-mysql-ms-67d7f88784-5r56x           1/1     Running     0             9m10s
upm-control-nacos-0                             1/1     Running     0             9m10s
upm-control-nacos-init-db-ftdhw                 0/1     Completed   0             9m10s
upm-control-nginx-6bc6845cff-zwrvk              1/1     Running     0             9m9s
upm-control-operatelog-6846b65694-pjjv5         1/1     Running     0             9m10s
upm-control-postgresql-ms-cc84f49d6-5f55k       1/1     Running     0             9m10s
upm-control-redis-cluster-ms-5bd595dcd-6nxb5    1/1     Running     0             9m9s
upm-control-redis-master-0                      1/1     Running     0             9m10s
upm-control-redis-ms-6b97895775-rl27w           1/1     Running     0             9m10s
upm-control-resource-7cf4ffc848-zzwsp           1/1     Running     0             9m10s
upm-control-ui-57ff64f67-trpw9                  1/1     Running     0             9m10s
upm-control-user-5cc4c6b555-snmtk               1/1     Running     0             9m10s
upm-control-zookeeper-ms-6d8b7f8f44-v5k2c       1/1     Running     0             9m10s
```

Activate cluster registration to support incluster mode. (Required for single cluster)

```
$ sh run.sh v1.2.2
$ kubectl apply -f yaml/upm/clusterrolebinding.yaml 
clusterrolebinding.rbac.authorization.k8s.io/upm-system-admin-default-account created
```

### 6.2 Install upm-engine

set up`upm-engine`The configuration file, the example content is as follows:

```
$ vi upm-release-v1.2.2/scripts/engine/yaml/upm/env.yaml
imageRegistry: "quay.io"
upm-engine:
  nodeNames: "upm-platform01"
```

`upm-engine`Required parameter description for component installation:

* `imageRegistry`: Configure the container mirror repository name.
* `upm-engine.nodeNames`: Deploy the upm-engine scheduling to the specified node.

Execute the installation script:

```
$ cd upm-release_v1.2.2/scripts/engine/
$ sh upm-install.sh v1.2.2
```

Finally, check`upm-system`Pod status of all components.

```
$ kubectl get pods -n upm-system | grep engine
upm-engine-kauntlet-675b45d44b-nb6xd        1/1     Running   0          6m40s 
upm-engine-tesseract-cube-c8d6d474f-mjd4f   1/1     Running   0          6m40s 
```

### 6.3 Install cert-manager

Cert-manager is related to the creation of certificate authentication by the Elasticsearch service.

set up`cert-manager`The configuration file, the example content is as follows:

```
vi upm-release-v1.2.2/scripts/engine/yaml/cert-manager/env.yaml
imageRegistry: "quay.io"
cert-manager:
  nodeNames: "upm-platform01"
```

`cert-manager`Required parameter description for component installation:

* `imageRegistry`: Configure the container mirror repository name.
* `cert-manager.nodeNames`: Deploy the cert-manager schedule to the specified node.

Execute the installation script:

```
$ cd upm-release_v1.2.2/scripts/engine/
$ sh cert-manager-install.sh  v1.2.2
```

examine`cert-manager`Pod status of all components.

```
$ kubectl get pod -n cert-manager | grep cert-manager
cert-manager-cainjector-774b684f98-gsrqd   1/1     Running   0          3m59s
cert-manager-controller-8457454ffc-trrdm   1/1     Running   0          3m59s
cert-manager-webhook-5bfb6df4bf-9nxwn      1/1     Running   0          3m59s
```

### 6.4 Install cnpg-operator

Execute the installation script:

```
$ sh upm-release_v1.2.2/scripts/engine/cnpg-install.sh v1.2.2
```

examine`cnpg-operator`Pod status of all components.

```
$ kubectl get pod -n cnpg-system 
NAME                                       READY   STATUS    RESTARTS   AGE
cnpg-controller-manager-7fd67898bb-dht52   1/1     Running   0          40s
```

## 7. Access the interface

* UPM UI format: http://\[master\_ip]:\[port]/upm-ui/#/login;

Example:

* http://172.168.24.61:32010/upm-ui/#/login
* Default user: super\_root
* Default password: Upm@2024!
