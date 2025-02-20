# Concepts

This glossary provides definitions and explanations of important terms and concepts used in UPM.\


### Project

A project is an annotated Kubernetes namespace that is the core tool for managing access to resources for ordinary users. A project allows users and manages resource object content, isolated from other users' resources. Users must have administrator access to the project, and if they are allowed to create the project, they automatically have access to their own project.Projects can have a unique name, display name, and description.

* The unique name is the Unique Device Identifier of the project, which is most obvious when using CLI tools or APIs. The maximum length of the name is 63 characters.
* The optional displayName is how the project is displayed in the network console (default is name).
* Optional descriptions can be more detailed descriptions of the project, which can be viewed in the UI Console.

\


### Cluster

Cluster refers to the Kubernetes cluster that can be allocated resources. The cluster object information is registered in the UPM API-Server. The UPM API-Server will automatically collect its information and save it to the corresponding data database & table on the UPM API-Server.Clusters can have a unique name, display name, available Kubernetes Service type, authentication mode, and description.

* The unique name is the Unique Device Identifier of the cluster, which is most obvious when using CLI tools or APIs. The maximum length of the name is 63 characters.
* Display name is the display method of the project in the network Console (with a unique name filled in by default), optional.
* Available Kubernetes Service type refers to the Kubernetes Service type available to the cluster. Kubernetes Service is the method of exposing network applications running on one or a group of [Pods ](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/)as network services. Each Service object defines a logical set of endpoints (usually these endpoints are Pods) and policies for how to access these Pods. Available `type` values and their behaviors are:
  * **`ClusterIP exposes`** the Service through the internal IP of the cluster. When this value is selected, the Service can only be accessed within the cluster. This is also the default value used when you do not explicitly specify the `type` of the service. You can use [Ingress ](https://kubernetes.io/zh-cn/docs/concepts/services-networking/ingress/)or [Gateway API ](https://gateway-api.sigs.k8s.io/)to expose the service to the public Internet.
  * [**`NodePort`** ](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#type-nodeport)exposes the Service through the IP and static port ( `NodePort` ) on each node. To make the Service accessible through the node port, Kubernetes configures a cluster IP address for the Service, which is equivalent to requesting a service with `type: ClusterIP` .
  * [**`LoadBalancer`** ](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#loadbalancer)exposes the Service externally using the Cloud Computing Platform's load balancer. Kubernetes do not provide a load balancing component directly; you must either provide one or integrate your Kubernetes cluster with a Cloud Computing Platform.
* Authentication mode refers to the type of access authentication information Kubernetes cluster, optional types are **`kubeconfig`** and **`Bearer token`** .
  * **`Kubeconfig`** : The file used to configure cluster access is called **the kubeconfig file**
  * **`Bearer token`** : is a widely used authentication mechanism, widely used in Web API and other Web services. Token is used for API access authentication, located in API Header Authorization. Bearer < token >.
* The description can be a more detailed description that can be viewed in the UI Console, optional.

\


### Regional Zone

Region is a logical object of resource scope that UPM will Kubernetes in the cluster. It is an abstract grouping of resource distribution. When a user specifies a resource range, it does not need to be related to the specific location of the allocated server in the same data center, but it needs to be in the same region. Region provides a way to express fault tolerance and infrastructure isolation between resources.\


### Software Software

The software is a database type and corresponding version supported by UPM. It includes software configuration templates, configuration parameter descriptions, and Pod extension content. Administrators and users can manage content through the software to unify the configuration file content and Pod extension content.\


### **Node** Node

Nodes are the vectors of UPM's eventual running workloads and the core resource objects in the Kubernetes, Kubernetes execute [workloads ](https://kubernetes.io/zh-cn/docs/concepts/workloads/)by placing containers in pods that run on nodes (Nodes). Nodes can be either virtual machines or physical machines, depending on the cluster configuration. Each node contains the services required to run [pods ](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/); these nodes are managed by the [control plane ](https://kubernetes.io/zh-cn/docs/reference/glossary/?all=true#term-control-plane).\


### Node Group

Node groups are used to group nodes and can be used to limit what types of software can be allocated in a given set of nodes. Example use cases for node pools include segmenting nodes by environment (development, staging, production), department (engineering, finance, support), or function (database, ingress agent, application).\


### Unit Unit

Unit is the process of filling items in a way that maximizes the use of boxes. This extends to Nomad, where the Client is the "trash can" and the project is the task group. Nomad optimizes resources by efficiently packaging tasks to the Client's computer.Unit (like a block on a Rubik's cube) is the sum of resource objects required by a service instance. Unit contains a [Pod ](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/), a configuration file used by the running service, a persistent[storage volume ](https://kubernetes.io/zh-cn/docs/concepts/storage/persistent-volumes/)for storing data usage; Unit integrates and utilizes the necessary resources contained in the service operation and maintenance in a complete service instance lifecycle management manner.\


### Unit group UnitSet

UnitSet is a workload API object used to manage general-purpose stateful database instances.UnitSet is used to manage the deployment and scaling of Unit collections, and provides persistent storage and identifiers, configuration file templates, encryption information, authentication certificates, and rolling update policies for these Units. UnitSet manages a group of Units based on a set of Pod specifications of the same type.\


### Ticket Order

Ticket is the definition and description specification of the expected database and Middleware service provided by the user, which is used to declare the workload of multiple and different types of UnitSets. Ticket is a form of _expected state_ ; the user expresses how a set of complex cluster architecture databases or Middleware services run, how to manage configurations, and how to extend additional CRD support for advanced operation and maintenance operations.\


### Workload **Services** Service

The workload service is the smallest set of workflow task units in UPM. UPM API-Server automatically resolves tasks into Kubernetes APIs by obtaining task definition descriptions. CR (such as unit group UnitSet) is executed by the controller of the Kubernetes cluster, which enables UPM to flexibly support the load architecture of multiple databases it supports and quickly expand new task flows to meet customer needs.\


### Workload Service Group

The workload service group is a set of workflow definitions in UPM. UPM API-Server automatically splits the workflow into multiple workflow task definitions by obtaining the definition description of the ticket. The tasks will be converted into multiple driver modes for execution, and the CR of the Kubernetes API will be executed by the controller of the Kubernetes cluster. This enables UPM to flexibly support the load architecture of multiple databases it supports and quickly expand new task flows to meet customer needs.\
