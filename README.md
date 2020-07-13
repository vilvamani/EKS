Contents:

* [Intent](#markdown-header-intent)
* [Architecture Overview](#markdown-header-architecture-overview)
* [Requirements](#markdown-header-requirements)
* [Recommendations](#markdown-header-recommendations)
* [Configuration Review & Deployment](#markdown-header-configuration-review-deployment)
* [Recommended Deployment Additions - Autoscaling](#markdown-header-recommended-deployment-additions-autoscaling)
* [Recommended Deployment Additions - Bastion Host](#markdown-header-recommended-deployment-additions-bastion-host)
* [Deployment Complete](#markdown-header-deployment-complete)
* [Coordinating Notes](#markdown-header-coordinating-notes)
--------------------------------------------------------------------------------------

# Kubernetes Reference Architecture - Boomi Molecule

## Intent
The following reference architecture outlines the requirements and limitations associated with running a Boomi Molecule in a Kubernetes based containerized environment.

Kubernetes (K8s) is a powerful and complex container orchestration platform with an ever expanding ecosystem of services, support and tools.  The introduction and general education of Kubernetes in its many forms and possible configuration permutations are outside the scope of this document.  As such, **it is expected that the user has a working knowledge of Kubernetes concepts and components with an operational Kubernetes environment already deployed.**

This reference document provides a working Amazon Elastic Kubernetes Service (EKS) example in which the user can expand upon to reach a desired Kubernetes configuration state for their particular use case.

--------------------------------------------------------------------------------------

## Architecture Overview
This reference architecture utilizes a containerized deployment in Amazon EKS - Managed Kubernetes Service to convey Boomi Molecule Kubernetes configuration requirements.  The example architecture deploys an external load balanced, highly available, Kubernetes containerized Molecule cluster with the ability to dynamically scale both cluster nodes and pods.

![Boomi_EKS.png](image/Boomi_EKS.png)

--------------------------------------------------------------------------------------

## Known Limitations
**Atom Queues** are not currently supported in Boomi Molecule Clusters that wish to utilize the elasticity function in a horizontally scaled environment like EKS. Specifically, scaling down with an Atom Queue deployed will result in data loss.

--------------------------------------------------------------------------------------

## Requirements
#### Kubernetes Version
The Boomi Molecule and this reference architecture requires **Kubernetes version 1.16** or greater.  At the time this document was created, **EKS 1.16** is available and meets the requirement.

#### EKSCTL
This reference architecture requires the command-line utility **eksctl** to be installed for the example deployment.  Installation instructions can be found [here](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html).

#### KUBECTL
This reference architecture requires the Kubernetes command-line utility **kubectl** to be installed for the example deployment.  Installation instructions can be found [here](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html).

#### Kubernetes Configuration
Below are Kubernetes configuration requirements associated with successfully running a Boomi Elastic Molecule in a Kubernetes Cluster.  These configuration components are expanded on later in the document during review of the accompanying yaml configuration.  It is important for a user to familiarize themselves with the functionality of each before moving forward.

* [Kubernetes Cluster](https://kubernetes.io/docs/concepts/overview/components/)
* [Node(s)](https://kubernetes.io/docs/concepts/architecture/nodes/)
* [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
* [Kubernetes Service](https://kubernetes.io/docs/concepts/services-networking/service/)
* [Storage Class](https://kubernetes.io/docs/concepts/storage/storage-classes/)
* [Persistent Volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
* [Persistent Volume Claim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
* [Stateful Set](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
    * The most impactful Kubernetes configuration requirement for a Boomi Molecule is the StatefulSet.  StatefulSets require configuration consideration to effectively utilize AWS autoscaling functions with multiple Availability Zones (AZ) to achieve High Availability (HA) and system resiliency.

#### Boomi Molecule Configuration / Infrastructure
Below are general requirements associated with the deployment of a Boomi Molecule Cluster.

* Boomi account credentials.
* NFS solution provisioned and reachable by the Kubernetes environment.
    * The Boomi Molecule requires the provisioning and availability of a Network File System (NFS).
* Boomi Molecule Docker image ([publicly available](https://hub.docker.com/r/boomi/molecule)).

--------------------------------------------------------------------------------------

## Recommendations
Below are Kubernetes configuration recommendations for value added Kubernetes functions.

#### Kubernetes Configuration
**Autoscaling**  
The Boomi Molecule shows its true power through its elastic capabilities. Although it is possible to deploy the Boomi Molecule in a Kubernetes containerized environment without taking advantage of elasticity, it is highly recommended that users do so by deploying the following items:

* [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md)
* [Metrics Server](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-metrics-pipeline/#metrics-server)
* [Horizontal Pod Autoscaler (HPA)](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

--------------------------------------------------------------------------------------
## Configuration Review & Deployment
The following configuration component files each represent a critical component in the example containerized stack, and are explained in depth later in this document.

* boomi_molecule_eks_cluster.yaml
* boomi_molecule_eks_storageclass.yaml
* boomi_molecule_eks_pv.yaml
* boomi_molecule_eks_pvclaim.yaml
* boomi_molecule_eks_secret.yaml
* boomi_molecule_eks_service.yaml
* boomi_molecule_eks_statefulset.yaml
* boomi_molecule_eks_ca.yaml
* k8s_metrics_server_install.sh
* boomi_molecule_eks_hpa.yaml

These files correspond directly to an ordered deployment and are used within the following steps:

Required

1. Create the [Kubernetes Cluster and Node(s)](#markdown-header-kubernetes-cluster-nodes).
    * In the deployment example, we will use the eksctl CLI tool for creating clusters in Amazon EKS.
1. Create the required [NFS](#markdown-header-create-the-required-nfs-for-the-boomi-molecule) for the Boomi Molecule.
    * In the deployment example we will utilize Amazon’s Elastic File System (EFS) to meet this requirement.
1. Create [Storage Class]()
1. Create [PersistentVolume](#markdown-header-create-persistentvolume).
1. Create [PersistentVolumeClaim](#markdown-header-create-persistentvolumeclaim).
1. Create Kubernetes [Secret](#markdown-header-create-kubernetes-secret).
1. Create [Service](#markdown-header-create-service).
1. Create [StatefulSet](#markdown-header-create-statefulset).

Recommended for utilizing autoscaling/elasticity capabilities:

1. Deploy the [Cluster Autoscaler]()
1. Deploy the Kubernetes [Metrics Server](#markdown-header-deploy-the-kubernetes-metrics-server).
1. Deploy [Horizontal Pod Autoscaler (HPA)](#markdown-header-deploy-horizontal-pod-autoscaler-hpa).

Ancillary:

1. Deploy the Bastion Host.

--------------------------------------------------------------------------------------
#### Kubernetes Cluster & Node(s)
In the boomi_molecule_eks_cluster.yaml configuration file, we define the target cluster and required node groups for deployment.  The aforementioned StatefulSet requirement coupled with the desire to utilize multiple AWS Availability Zones (AZ) dictates the creation of a node group per AZ.

To account for this requirement, we explicitly define our desired AZs (us-east-1a, us-east-1b, us-east-1c) that correspond with the delineated metadata.region element.

We also utilize EKS [managedNodeGroups](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html), which provide a number of benefits including the automation of autoscaling requirements.

```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: EKS-boomi-molecule-main
  region: us-east-1
  version: "1.16"

vpc:
  cidr: 10.100.0.0/16

# cluster AZs must be set explicitly for StatefulSet nodegroup per AWS AZ requirement
availabilityZones: ["us-east-1a", "us-east-1b", "us-east-1c"]

cloudWatch:
  clusterLogging:
    # all supported types: "api", "audit", "authenticator", "controllerManager", "scheduler"
    enableTypes: ["all"]

managedNodeGroups:
  - name: us-east-1a
    minSize: 1
    maxSize: 100
    desiredCapacity: 1
    volumeSize: 20
    # scope node to single AZ, best practice for StatefulSet deployment.
    availabilityZones: ["us-east-1a"]
    privateNetworking: true
    ssh:  # use existing EC2 key.
      publicKeyName: <my_ssh_key_in_aws>
    iam:
      withAddonPolicies:
        autoScaler: true
        cloudWatch: true
        efs: true


  - name: us-east-1b
    minSize: 1
    maxSize: 100
    desiredCapacity: 1
    volumeSize: 20
    availabilityZones: ["us-east-1b"]
    privateNetworking: true
    ssh:
      publicKeyName: <my_ssh_key_in_aws>
    iam:
      withAddonPolicies:
        autoScaler: true
        cloudWatch: true
        efs: true

  - name: us-east-1c
    minSize: 1
    maxSize: 100
    desiredCapacity: 1
    volumeSize: 20
    availabilityZones: ["us-east-1c"]
    privateNetworking: true
    ssh:
      publicKeyName: <my_ssh_key_in_aws>
    iam:
      withAddonPolicies:
        autoScaler: true
        cloudWatch: true
        efs: true
```

Configuration elements of note:
```
managedNodeGroups.privateNetworking: true
```
* Deploys cluster nodes in private subnets.  See the [Recommended Deployment Additions - Bastion Host](#markdown-header-recommended-deployment-additions-bastion-host) section for more information.

```
managedNodeGroups.ssh.publicKeyName:
```
* Utilizes an established SSH key in AWS during creation of Cluster Nodes.

```
managedNodeGroups.iam.withAddonPolicies.autoScaler: true
```
* Creates required IAM policies for cluster autoscaling as well as applies required asset tagging to facilitate auto discovery.

```
cloudWatch.clusterLogging.enableTypes: ["all"]
managedNodeGroups.iam.withAddonPolicies.cloudWatch: true
```
* Creates required IAM policies for cluster logging to CloudWatch.
* Creates CloudWatch log group with "api", "audit", "authenticator", "controllerManager" and "scheduler" log streams.
```
managedNodeGroups.iam.withAddonPolicies.efs: true
```
* Creates required IAM policies for cluster access to Amazon EFS used to fulfill Boomi Molecule NFS requirements.


The EKS cluster and node groups are created using the following eksctl command:
```
$ eksctl create cluster -f boomi_molecule_eks_cluster.yaml
```

Expected return:
```
[✔]  EKS cluster "EKS-boomi-molecule-main" in "us-east-1" region is ready
```

--------------------------------------------------------------------------------------
#### Create the required NFS for the Boomi Molecule
The Boomi Molecule requires the provisioning and availability of a Network File System (NFS).  In the deployment example, we will utilize **Amazon's Elastic File System (EFS)** to meet this requirement.  Detailing the creation of an Amazon EFS is out of scope for this document.  More information on Amazon EFS can be found [here](https://docs.aws.amazon.com/efs/latest/ug/whatisefs.html).

The [AWS EFS Container Storage Interface](https://github.com/kubernetes-sigs/aws-efs-csi-driver) (CSI) driver must be deployed to the Cluster to integrate with the created Amazon EFS.  The driver is referenced by the Storage Class, and Persistent Volume configuration files.

The EFS CSI is deployed to the Cluster using the following kubectl command:
```
$ kubectl apply -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"
```

--------------------------------------------------------------------------------------
#### Create StorageClass
A Kubernetes [Storage Class](https://kubernetes.io/docs/concepts/storage/storage-classes/) is essentially a storage blueprint that abstracts away the underlying storage provider, as well as other parameters, like disk-type (e.g.; solid-state vs standard disks).

The Storage Class is defined in the [boomi_molecule_eks_storageclass.yaml](config/boomi_molecule_eks_storageclass.yaml) configuration file:
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
```

We can see the EFS CSI identified corresponding to the NFS/EFS requirement and the driver deployment delineated in the Cluster configuration.

The StorageClass is deployed to the created Cluster using the following kubectl CLI command:
```
$ kubectl apply -f  boomi_molecule_eks_storageclass.yaml
```

Expected return:
```
storageclass.storage.k8s.io/efs-sc created
```

--------------------------------------------------------------------------------------
#### Create PersistentVolume
A [PersistentVolume (PV)](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) is a piece of storage in the cluster. It is a resource in the cluster just like a node is a cluster resource. PVs have a lifecycle independent of any individual Pod that uses the PV.  The PV is bound to the external NFS created previously.  The PV is the first of two configuration components that account for the Boomi Molecule NFS requirement, taking the form of Persistent Storage in a Kubernetes deployment. The required PV is defined in the [boomi_molecule_eks_pv.yaml](config/boomi_molecule_eks_pv.yaml) configuration file:

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: <efs_file_system_id>
```

Configuration elements of note:
```
spec.capacity.storage:
```
* This value should be configured for your specific storage requirements.

```
spec.csi.volumeHandle: <efs_file_system_id>
```
* The EFS file system ID from the previous Amazon EFS creation step is required.

The PersistentVolume is deployed to the created Cluster using the following kubectl CLI command:
```
$ kubectl apply -f  boomi_molecule_eks_pv.yaml
```

Expected return:
```
persistentvolume/efs-pv created
```

--------------------------------------------------------------------------------------
#### Create PersistentVolumeClaim
A [PersistentVolumeClaim (PVC)](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) is a request for storage.  PVCs consume PV resources (like the one created in the previous step).  The PVC is the second configuration component that accounts for the Boomi Molecule NFS requirement.  The required PVC is defined in the [boomi_molecule_eks_pvclaim.yaml](config/boomi_molecule_eks_pvclaim.yaml) configuration file and is later referenced in the StatefulSet configuration.
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```
Configuration elements of note:
```
spec.resources,requests.storage
```
* This value should be configured for your specific storage requirements and should match the value delineated in the boomi_molecule_eks_pv.yaml.

The PersistentVolumeClaim is deployed to the Cluster using the following kubectl CLI command:
```
$ kubectl apply -f  boomi_molecule_eks_pvclaim.yaml
```

Expected return:
```
persistentvolumeclaim/efs-claim created
```

--------------------------------------------------------------------------------------
#### Create Kubernetes Secret
Kubernetes provides an object called [Secret](https://kubernetes.io/docs/concepts/configuration/secret/), which you can use to store sensitive data and provides more control over access and usage of passwords, keys, etc.

The Boomi Molecule Docker image expects a number of environment variables passed during container creation.  Boomi Account_ID, Username, and Password are expected variables that represent sensitive information.  The example deployment utilizes a Kubernetes Secret to provide access to sensitive information rather than putting it verbatim in a Pod definition.  The StatefulSet configuration references the Kubernetes Secret.  The Kubernetes Secret is defined in the [boomi_molecule_eks_secret.yaml](config/boomi_molecule_eks_secret.yaml).
```
apiVersion: v1
kind: Secret
metadata:
  name: boomi-secret
type: Opaque
stringData:
  username: **************************
  password: **************************
  account: *************************
```
**Note:**  YAML escape requirements for special characters must be observed in the ```stringData``` fields for ```stringData.password```, ```stringData.username``` and ```stringData.account```.  For example a password like:  
```
My"crazy',pa#$wo\rd"1!
```
would require the following escaping with encapsulated double quotation marks:
```
"My\"crazy',pa#$wo\\rd\"1!"
```

The Secret is deployed to the Cluster using the following kubectl CLI command:
```
$ kubectl apply -f  boomi_molecule_eks_secret.yaml
```
Expected return:
```
secret/boomi-secret created
```

##### Alternative Configuration - Secret
The Boomi Molecule also supports an [Installer Token](https://help.boomi.com/bundle/integration/page/r-atm-Installer_Token_object_8bb87600-5048-4826-b716-f0173ca8798d.html) object for installation instead of the above username and password credential method.

--------------------------------------------------------------------------------------
#### Create Service
A [Kubernetes Service](https://kubernetes.io/docs/concepts/services-networking/service/) is an abstract way to expose an application running on a set of Pods (Molecule Nodes) as a network service.
The required Kubernetes Service is defined in the [boomi_molecule_eks_service.yaml](config/boomi_molecule_eks_service.yaml) configuration file.
```
apiVersion: v1
kind: Service
metadata:
  name: molecule-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
  labels:
    app: molecule
spec:
  selector:
    app: molecule
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9090
```

Additional configuration elements of note:
```
metadata.annotations.service.beta.kubernetes.io/aws-load-balancer-type:"nlb"
```
* This annotation directs the creation of a Network Load Balancer (NLB) when the service is deployed with type: LoadBalancer.

The Service is deployed to the created Cluster using the following kubectl CLI command:
```
$ kubectl apply -f  boomi_molecule_eks_service.yaml
```
Expected return:
```
service/molecule-service created
```

--------------------------------------------------------------------------------------
#### Create StatefulSet
In short, a [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) is:
>"A workload API object used to manage stateful applications, the deployment and scaling of a set of Pods, and provides guarantees about the ordering and uniqueness of these Pods."

The Boomi Molecule requires the use of a StatefulSet to provide for the following:

* Stable, persistent storage.
* Ordered, graceful deployment and scaling.

The Stateful Set is defined in the [boomi_molecule_eks_statefulset.yaml](config/boomi_molecule_eks_statefulset.yaml).  This configuration file details a number of critical elements and ties together all previous configuration components into deployed Pods.
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: molecule
  labels:
    app: molecule
spec:
  selector:
    matchLabels:
      app: molecule
  serviceName: "molecule-service"
  replicas: 3
  template:
    metadata:
      labels:
        app: molecule
    spec:
      terminationGracePeriodSeconds: 60
      volumes:
        - name: molecule-storage
          persistentVolumeClaim:
            claimName: efs-claim
        - name: tmpfs
          emptyDir: {}
        - name: cgroup
          hostPath:
            path: /sys/fs/cgroup
            type: Directory
      containers:
      - image: boomi/molecule:release
        imagePullPolicy: Always
        name: atom-node
        ports:
        - containerPort: 9090
          protocol: TCP
        lifecycle:
          preStop:
            exec:
              command:
                - sh
                - /home/boomi/scaledown.sh
        resources:
          limits:
            cpu: "1000m"
            memory: "1536Mi"
          requests:
            cpu: "500m"
            memory: "1024Mi"
        volumeMounts:
          - mountPath: "/mnt/boomi"
            name: molecule-storage
          - name: tmpfs
            mountPath: "/run"
          - name: tmpfs
            mountPath: "/tmp"
          - name: cgroup
            mountPath: /sys/fs/cgroup
        startupProbe:
          failureThreshold: 60
          exec:
            command:
              - sh
              - /home/boomi/probe.sh
              - startup
        readinessProbe:
          periodSeconds: 10
          initialDelaySeconds: 10
          exec:
            command:
              - sh
              - /home/boomi/probe.sh
              - readiness
        livenessProbe:
          periodSeconds: 60
          exec:
            command:
              - sh
              - /home/boomi/probe.sh
              - liveness
        env:
        - name: BOOMI_ATOMNAME
          value: "boomi-molecule-eks"
        - name: ATOM_LOCALHOSTID
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: BOOMI_ACCOUNTID
          valueFrom:
            secretKeyRef:
              name: boomi-secret
              key: account
        - name: URL
          value: "https://starlord.sandbox.boomi.com"
        - name: BOOMI_USERNAME
          valueFrom:
            secretKeyRef:
              name: boomi-secret
              key: username
        - name: BOOMI_PASSWORD
          valueFrom:
            secretKeyRef:
              name: boomi-secret
              key: password
        - name: BOOMI_CONTAINERNAME
          value: "boomi-molecule-eks"
        - name: INSTALLATION_DIRECTORY
          value: "/mnt/boomi"
        - name: ATOM_VMOPTIONS_OVERRIDES
          value:
        - name: CONTAINER_PROPERTIES_OVERRIDES
          value:
```

Configuration elements of note:
```
spec.serviceName: "molecule-service"
```
* Links Stateful Set to previously deployed Service.

```
spec.template.spec.terminationGracePeriodSeconds: 60
```
* Duration in seconds the pod needs to terminate gracefully after the PreStop hook is called.  This number should be greater than the Molecule setting for com.boomi.container.elasticity.force RestartOverride.  More information on the setting can be found [here](https://help.boomi.com/bundle/integration/page/r-atm-Properties_panel_Advanced_tab_c39737e8-1b16-4fdd-b414-152694364c14.html).

```
spec.template.spec.volumes.persistentVolumeClaim.claimName: efs-claim
spec.template.spec.volumesMounts.name: molecule-storage
```
* References previously deployed Persistent Volume Claim, which in turn references the Persistent Claim utilizing the EFS CSI driver to interface with the deployed Amazon EFS culminating in a mount point in deployed Pods.

```
spec.template.spec.volumes.name: tmpfs
spec.template.spec.volumes.name: cgroup
spec.template.spec.volumesMounts.name: tmpfs
spec.template.spec.volumesMounts.name: cgroup
```
* Volume configuration allowing for unprivileged container security context.

```
spec.template.spec.containers.image: boomi/molecule:release
```
* Pulls Boomi Molecule release docker image directly from Docker Hub.

```
spec.template.spec.containers.resources.limits.cpu: “1000m”
spec.template.spec.containers.resources.limits.memory: “1024Mi”
spec.template.spec.containers.resources.requests.cpu: “500m”
spec.template.spec.containers.resources.requests.memory: "768Mi"
```
* Pod resource allocations used in scheduling and Horizontal Pod Autoscaling through the Metrics Server.  The memory request value should be at least 20% above the heap size with a limit size 25% over the request value to avoid OOM kills.  The default heap size is set at 512MB.  The heap size can be overridden using the vm options override environment variable.

```
spec.template.spec.containers.startupProbe:
spec.template.spec.containers.readinessProbe:
spec.template.spec.containers.livenessProbe:
```
* Provides health checks for initial and continued Molecule pod status.

```
spec.template.spec.containers.env.name: BOOMI_ACCOUNTID
spec.template.spec.containers.env.name: BOOMI_USERNAME
spec.template.spec.containers.env.name: BOOMI_PASSWORD
```
* Sensitive information environment variables derived from previously deployed Kubernetes Secret.

```
spec.template.spec.containers.env.name: ATOM_VMOPTIONS_OVERRIDES
spec.template.spec.containers.env.name: CONTAINER_PROPERTIES_OVERRIDES
```
* A | (pipe) separated list of vm options and container properties to set on a new installation.  More information can be found in the image Overview on Docker Hub [here](https://hub.docker.com/r/boomi/molecule).

The Stateful Set is deployed to the Cluster using the following kubectl CLI command:
```
$ kubectl apply -f  boomi_molecule_eks_statefulset.yaml
```

Expected return:
```
statefulset.apps/molecule created
```

--------------------------------------------------------------------------------------
## Recommended Deployment Additions - Autoscaling

#### Deploy the Cluster Autoscaler (CA)
The [Kubernetes Cluster Autoscaler](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md) (CA) automatically adjusts the number of nodes in the cluster when pods fail to launch due to lack of resources, or when nodes in the cluster are underutilized and the pods are rescheduled onto other nodes in the cluster.

In the [boomi_molecule_eks_ca.yaml](config/boomi_molecule_eks_ca.yaml) configuration file we define and deploy the Cluster Autoscaler.  A large majority of the configuration sets up the correct ServiceAccount, Roles and RBAC allocations required by the Cluster Autoscaler.  Critical parameters associated with scaling behavior are found in the Deployment document (kind: Deployment), inside the element. ```spec.template.spec.containers.command:```.
```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
  name: cluster-autoscaler
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-autoscaler
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
rules:
  - apiGroups: [""]
    resources: ["events", "endpoints"]
    verbs: ["create", "patch"]
  - apiGroups: [""]
    resources: ["pods/eviction"]
    verbs: ["create"]
  - apiGroups: [""]
    resources: ["pods/status"]
    verbs: ["update"]
  - apiGroups: [""]
    resources: ["endpoints"]
    resourceNames: ["cluster-autoscaler"]
    verbs: ["get", "update"]
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["watch", "list", "get", "update"]
  - apiGroups: [""]
    resources:
      - "pods"
      - "services"
      - "replicationcontrollers"
      - "persistentvolumeclaims"
      - "persistentvolumes"
    verbs: ["watch", "list", "get"]
  - apiGroups: ["extensions"]
    resources: ["replicasets", "daemonsets"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["policy"]
    resources: ["poddisruptionbudgets"]
    verbs: ["watch", "list"]
  - apiGroups: ["apps"]
    resources: ["statefulsets", "replicasets", "daemonsets"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses", "csinodes"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["batch", "extensions"]
    resources: ["jobs"]
    verbs: ["get", "list", "watch", "patch"]
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["create"]
  - apiGroups: ["coordination.k8s.io"]
    resourceNames: ["cluster-autoscaler"]
    resources: ["leases"]
    verbs: ["get", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["create","list","watch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["cluster-autoscaler-status", "cluster-autoscaler-priority-expander"]
    verbs: ["delete", "get", "update", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-autoscaler
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-autoscaler
subjects:
  - kind: ServiceAccount
    name: cluster-autoscaler
    namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: cluster-autoscaler
subjects:
  - kind: ServiceAccount
    name: cluster-autoscaler
    namespace: kube-system

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
  annotations:
    cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
      annotations:
        prometheus.io/scrape: 'true'
        prometheus.io/port: '8085'
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
        - image: us.gcr.io/k8s-artifacts-prod/autoscaling/cluster-autoscaler:v1.16.5
          name: cluster-autoscaler
          resources:
            limits:
              cpu: 100m
              memory: 300Mi
            requests:
              cpu: 100m
              memory: 300Mi
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --balance-similar-node-groups=true
            - --skip-nodes-with-system-pods=false
            - --scale-down-unneeded-time=1m
            - --scale-down-delay-after-add=1m
            - --scale-down-delay-after-failure=1m
            - --scan-interval=5s
            - --max-nodes-total=300
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/IK-EKS-CA-HPA-molecule-cluster-us-east-1
          volumeMounts:
            - name: ssl-certs
              mountPath: /etc/ssl/certs/ca-certificates.crt
              readOnly: true
          imagePullPolicy: "Always"
      volumes:
        - name: ssl-certs
          hostPath:
            path: "/etc/ssl/certs/ca-bundle.crt"
```

Configuration elements of note:
```
Deployment.spec.template.containers.command: expander=least-waste
```
* Required with more than one node group in the cluster.  There are three node groups, one per AZ due to the use of a StatefulSet in the deployment example.

```
Deployment.spec.template.containers.command: balance-similar-node-groups=true
```
* StatefulSet requirement for effective autoscaling.  This option instructs the Cluster Autoscaler to consider multiple autoscaling groups associated with each established Managed Node Group when creating new EC2 instances.

```
Deployment.spec.template.containers.command: scale-down-unneeded-time=1m
Deployment.spec.template.containers.command: scale-down-delay-after-add=1m
Deployment.spec.template.containers.command: scale-down-delay-after-failure=1m
Deployment.spec.template.containers.command: scan-interval=5s
```
* Timing intervals associated with scaling determination and event reception.  These are important to prevent thrashing during autoscaling.

```
Deployment.spec.template.containers.command: max-nodes-total=300
```
* Total number of EC2 instances that will be created inside the cluster.  The current value reflects the maximum number of allowed Cluster Nodes per Node Group (100) times the number of Node Groups (3).

```
node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/EKS-boomi-molecule-main
```
* Designates which Amazon resource tags identify EKS Cluster owned assets.  This is critical for dynamically identifying new Cluster assets when they come online.  **This value must match the delineated name of the Cluster in the ClusterConfig (metadata.name: <cluster_name>).**

The Cluster Autoscaler is deployed to the created Cluster using the following kubectl CLI command:
```
$ kubectl apply -f boomi_molecule_eks_ca.yaml
```

Expected return:
```
serviceaccount/cluster-autoscaler created
clusterrole.rbac.authorization.k8s.io/cluster-autoscaler created
role.rbac.authorization.k8s.io/cluster-autoscaler created
clusterrolebinding.rbac.authorization.k8s.io/cluster-autoscaler created
rolebinding.rbac.authorization.k8s.io/cluster-autoscaler created
deployment.apps/cluster-autoscaler created
```

--------------------------------------------------------------------------------------
#### Deploy the Kubernetes Metrics Server
The [Metrics Server](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-metrics-pipeline/#metrics-server) is a scalable, efficient source of container resource metrics for Kubernetes built-in autoscaling pipelines.  The Metrics Server collects resource metrics from Kubelets and exposes them in Kubernetes apiserver through a Metrics API for use by Horizontal Pod Autoscaler (HPA).  

The Metrics Server is a required component to deploy the HPA addressed later in this document.  The [k8s_metric_server_install.sh](config/k8s_metric_server_install.sh) file is a simple bash script that automates the download and deployment of the most current version of the Metrics Server.

Instructions on how to manually download and deploy the metric server can be found [here](https://github.com/kubernetes-sigs/metrics-server#deployment).

Expected return:
```
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
serviceaccount/metrics-server created
deployment.apps/metrics-server created
service/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
```


--------------------------------------------------------------------------------------
#### Deploy Horizontal Pod Autoscaler (HPA)
The Horizontal Pod Autoscaler (HPA) automatically scales the number of pods in a replication controller, deployment, replica set or stateful set based on observed CPU utilization and/or memory utilization.   As load inside the Boomi Molecule increases, Molecule nodes (Kubernetes pods) are added to the Kubernetes Cluster dynamically to provide capacity.  As load subsides, Molecule nodes (Kubernetes pods) are removed to reclaim resources.  

The HPA is described in the [boomi_molecule_eks_hpa.yaml](config/boomi_molecule_eks_hpa.yaml) configuration file.  The HPA is the initial catalyst for all autoscaling events inside the Cluster.
```
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: molecule-hpa
  labels:
    app: molecule
spec:
  scaleTargetRef:
    apiVersion: apps/v1beta1
    kind: StatefulSet
    name: molecule
  minReplicas: 3
  maxReplicas: 20
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 50
```


Configuration elements of note:
```
spec.minReplicas: 3
spec.maxReplicas: 20
```
* Minimum and maximum numbers of Pods maintained by the HPA.
* A minimum of 3 replicas is recommended.

```
spec.behavior.scaleDown.stabilizationWindowSeconds: 60
```
* Look back window for computed desired states when testing for a scaledown event.


```
spec.metrics
```
* Specifies which resource(s) to use when calculating the desired replica count (the maximum replica count across all metrics will be used).



The HPA is deployed to the Cluster using the following kubectl CLI command:
```
$kubectl apply -f boomi_molecule_eks_hpa.yaml --validate=false
```

Expected return:
```
horizontalpodautoscaler.autoscaling/molecule-hpa created
```

--------------------------------------------------------------------------------------
## Recommended Deployment Additions - Bastion Host
#### Deploy Bastion Host
Due to the fact that the Managed Node Groups are set to deploy their respective Cluster nodes into the established private subnets of the Cluster (ClusterConfig.managedNodeGroups.privateNetworking: true), SSH access to a given node is not publicly exposed.  As such an administrative bastion host is recommended should direct access to a given Cluster Node is required.  Deployment and configuration of the bastion host is outside the scope of this document.

It is important to clarify that this is for access to the underlying EC2 instance (Cluster Node) deployed in the EKS Cluster.  CLI access to pods is available regardless of the use of a bastion host via the ```kubectl exec -it <pod_name> -- /bin/bash``` command.

--------------------------------------------------------------------------------------
## Deployment Complete
At this point, all defined Amazon EKS components have been deployed.  Navigating to the Boomi AtomSphere Portal, we should see our Molecule reflected in the Atom Management section and waiting to be assigned to a given environment with subsequent process download for execution.  

As load inside the Boomi Molecule Nodes increases, Molecule Nodes (Kubernetes Pods) are added to the EKS Cluster dynamically across all Cluster Nodes to provide capacity.

When the Kubernetes Scheduler fails to deploy a given pod in response to a Horizontal Pod Autoscaler (HPA) request due to total Cluster Node resource limits, the Cluster Autoscaler (CA) deploys another Cluster Node (EC2 instance) providing more capacity.

As load subsides, Molecule Nodes (Kubernetes Pods) are removed or moved/consolidated to under utilized Cluster Nodes to reclaim resources.  As Cluster Nodes (EC2 instances) become inactive they are removed from the Cluster.

--------------------------------------------------------------------------------------
## Coordinating Notes
##### Useful kubectl CLI Commands
Below are some helpful kubectl and CLI commands to see Pod metrics, HPA events, Cluster Node metrics and CA events.

**Pod specific:**
HPA state
```
kubectl get hpa -w
```

Pod status/scaling
```
kubectl get pods -w -l app=molecule
```

Pod resource usage
```
watch kubectl top pods
```

**Cluster Node specific:**
CA log
```
kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler
```

Cluster Node resource usage
```
watch kubectl top nodes
```

Cluster Node status/Scaling
```
watch kubectl get nodes
```

Another good command that shows what Pods are on which Cluster Node:
```
kubectl get pod -o=custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName
```
