# Cluster Deployment Guide

This document will focus on how to deploy a **privatized Kubernetes cluster**-based cloud protosurvival separated distributed database, MatrixOne, from 0.

## Key steps

1. Deploying Kubernetes Clusters
2. Deployment Object Store MinIO  
3. Create and connect a MatrixOne cluster

## Noun interpretation

Since there are many Kubernetes related terms involved in this document, in order to allow you to understand the build process, here is a simple explanation of the important terms involved, if you need to know more about Kubernetes related content, you can refer directly to the [Kubernetes Chinese Community](http://docs.kubernetes.org.cn/)

- Pod

   Pods are the smallest resource management component in Kubernetes. Pods are also resource objects that minimize running containerized apps. A Pod represents a process running in a cluster. In a nutshell, we can call a set of apps that provide specific functionality a pod that will contain one or more container objects that together serve the outside world.

- Storage Class

   Storage Class, or **SC**, is used to label the characteristics and performance of a storage resource, which an administrator can define as a category, just as a storage device describes its own configuration (Profile). Depending on the description of the SC, the characteristics of the various storage resources can be visualized and the storage resources can be requested based on the application's demand for them.

- CSI

   Kubernetes provides a **CSI** interface (Container Storage Interface), based on which custom CSI plug-ins can be developed to support specific storage for decoupling purposes.

- PersistentVolume

   PersistentVolume, or **PV**, is a storage resource that includes settings for critical information such as storage capacity, access patterns, storage types, recycling policies, and backend storage types.

- PersistentVolumeClaim

   PersistentVolumeClaim, or **PVC**, is used as a user request for storage resources. It mainly includes the setting of information such as storage space request, access mode, PV selection criteria and storage category.

- Service

   Also called **SVC**, a mechanism for matching a set of pods to an external access service by way of label selection. Each svc can be understood as a microservice.

- Operator

   Kubernetes Operator is a way to encapsulate, deploy, and manage Kubernetes applications. We deploy and manage Kubernetes apps on Kubernetes using the Kubernetes API (Application Programming Interface) and kubectl tools.

## Deployment Architecture

### Dependent Components

The MatrixOne distributed system relies on the following components:

- Kubernetes: As a resource management platform for the entire MatrixOne cluster, including components such as Logservice, CN, TN, etc., run in Pods managed by Kubernetes. If a failure occurs, Kubernetes is responsible for weeding out the failed Pod and starting a new one to replace it.

- Minio: Provides object storage services for the entire MatrixOne cluster, with all MatrixOne data stored in object storage provided by Minio.

In addition, for container management and orchestration on Kubernetes, we need the following plugins:

- Helm:Helm is a package management tool for managing Kubernetes applications, similar to APT for Ubuntu and YUM for CentOS. It is used to manage preconfigured installation package resources called Charts.

- local-path-provisioner: As a plug-in in Kubernetes that implements the CSI (Container Storage Interface) interface, local-path-provisioner is responsible for creating persistent volumes (PVs) for the Pods and Minio of each component of MatrixOne for persistent storage of data.

### Overall architecture

The overall deployment architecture is shown in the following figure:

 <div align="center">
  <img src=https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-arch-overall.png?raw=true width=80% heigth=80%/>
 </div>

The overall architecture consists of the following components:

- The bottom layer is three server nodes: the first as host1, where the Kubernetes springboard is installed, the second as the Kubernetes master, and the third as the Kubernetes worker node.

- The upper layer is the installed Kubernetes and Docker environments that make up the cloud native platform layer.

- A layer of Kubernetes plug-ins managed based on Helm, including local-path-storage plug-ins that implement CSI interfaces, Minio, and MatrixOne Operator.

- The top layer is multiple Pods and Services generated by these component configurations.

### MatrixOne's Pod and Storage Architecture

MatrixOne creates a series of Kubernetes objects based on Operator's rules that are categorized by component and categorized into resource groups, CNSet, TNSet, and LogSet.

- Service: Services in each resource group need to be externally available through the Service. Service hosts the ability to connect to the outside world, ensuring service is still available if the Pod crashes or is replaced. The external application connects through the public port of the Service, while the Service forwards the connection to the appropriate Pod through internal forwarding rules.

- A containerized instance of the Pod:MatrixOne component that runs the core kernel code of MatrixOne.

- PVC: Each Pod declares its required storage resources through a PVC (Persistent Volume Claim). In our architecture, CN and TN need to request a storage resource as a cache, while LogService needs the corresponding S3 resource. These requirements are stated via PVC.

- PV:PV (Persistent Volume) is an abstract representation of a storage medium that can be viewed as a storage unit. After the PVC has been requested, the PV is created through software that implements the CSI interface and binds it to the PVC requesting the resource.

<div align="center">
<img src=https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-arch-pod.png?raw=true width=80% heigth=80%/>
</div>

## 1\. Deploy a Kubernetes cluster

Because the distributed deployment of MatrixOne relies on Kubernetes clusters, we need a Kubernetes cluster. This article will guide you through setting up a Kubernetes cluster by using **Kuboard-Spray**.

### Preparing the Cluster Environment

For clustered environments, you need to prepare as follows:

- 3 virtual machines
- The operating system uses CentOS 7.9 (needs to allow root remote login): two as machines to deploy Kubernetes and MatrixOne-related dependencies, and one as a springboard to build a Kubernetes cluster.
- extranet access conditions. All 3 servers require extranet mirror pull.

The distribution of individual machine conditions is as follows:

| **Host** | **Intranet IP** | **Extranet IP** | **mem** | **CPU** | **Disk** | **Role** |
| ------------ | ------------- | --------------- | ------- | ------- | -------- | ----------- |
| kuboardspray | 10.206.0.6 | 1.13.2.100 | 2G | 2C | 50G | hopper |
| master0 | 10.206.134.8 | 118.195.255.252 | 8G | 2C | 50G | master etcd |
| node0 | 10.206.134.14 | 1.13.13.199 | 8G | 2C | 50G | worker |

#### Springboard Deployment Kuboard Spray

Kuboard-Spray is a tool used to visually deploy Kubernetes clusters. It uses Docker to quickly pull up a web app that can visually deploy Kubernetes clusters. Once the Kubernetes cluster environment is deployed, you can stop the Docker app.

##### Springboard environment preparation

1. Install Docker: Since Docker is used, an environment with Docker is required. Install and start Docker on the springboard machine using the following command:

    ```
    curl -sSL https://get.docker.io/ | sh 
    #If in a restricted network environment in the country, you can change the following domestic mirror address 
    curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun 
    ```

2. Start Docker:

    ```
    [root@VM-0-6-centos ~]# systemctl start docker
    [root@VM-0-6-centos ~]# systemctl status docker
    ● docker.service - Docker Application Container Engine
       Loaded: loaded (/usr/lib/systemd/system/docker.service; disabled; vendor preset: disabled)
       Active: active (running) since Sun 2023-05-07 11:48:06 CST; 15s ago
         Docs: https://docs.docker.com
     Main PID: 5845 (dockerd)
        Tasks: 8
       Memory: 27.8M
       CGroup: /system.slice/docker.service
               └─5845 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

    May 07 11:48:06 VM-0-6-centos systemd[1]: Starting Docker Application Container Engine...
    May 07 11:48:06 VM-0-6-centos dockerd[5845]: time="2023-05-07T11:48:06.391166236+08:00" level=info msg="Starting up"
    May 07 11:48:06 VM-0-6-centos dockerd[5845]: time="2023-05-07T11:48:06.421736631+08:00" level=info msg="Loading containers: start."
    May 07 11:48:06 VM-0-6-centos dockerd[5845]: time="2023-05-07T11:48:06.531022702+08:00" level=info msg="Loading containers: done."
    May 07 11:48:06 VM-0-6-centos dockerd[5845]: time="2023-05-07T11:48:06.544715135+08:00" level=info msg="Docker daemon" commit=94d3ad6 graphdriver=overlay2 version=23.0.5
    May 07 11:48:06 VM-0-6-centos dockerd[5845]: time="2023-05-07T11:48:06.544798391+08:00" level=info msg="Daemon has completed initialization"
    May 07 11:48:06 VM-0-6-centos systemd[1]: Started Docker Application Container Engine.
    May 07 11:48:06 VM-0-6-centos dockerd[5845]: time="2023-05-07T11:48:06.569274215+08:00" level=info msg="API listen on /run/docker.sock"
    ```

Once the environment is ready, Kuboard-Spray can be deployed.

#### Deploy Kuboard-Spray

Install Kuboard-Spray by executing the following command:

```
docker run -d \
  --privileged \
  --restart=unless-stopped \
  --name=kuboard-spray \
  -p 80:80/tcp \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v ~/kuboard-spray-data:/data \
  eipwork/kuboard-spray:latest-amd64
```

If a mirror pull fails due to a network problem, you can use the following alternate address:

```
docker run -d \
  --privileged \
  --restart=unless-stopped \
  --name=kuboard-spray \
  -p 80:80/tcp \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v ~/kuboard-spray-data:/data \
  swr.cn-east-2.myhuaweicloud.com/kuboard/kuboard-spray:latest-amd64
```

Once this is done, you can enter `http://1.13.2.100` (Springboard IP address) in your browser to open the Kuboard-Spray web interface, enter the username `admin`, the default password `Kuboard123`, and log into the Kuboard-Spray interface as follows:

![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-1.png?raw=true)

Once logged in, you can begin deploying Kubernetes clusters.

### Visually deploy Kubernetes clusters

Once logged into the Kuboard-Spray interface, you can begin visual deployment of your Kubernetes cluster.

#### Import Kubernetes related resource packs

The installation interface downloads the resource package corresponding to the Kubernetes cluster via an online download to enable offline installation of the Kubernetes cluster.

1. Click on **Resource Pack Management** and select the appropriate version of Kubernetes Resource Pack to download:

    Download version `spray-v2.18.0b-2_k8s-v1.23.17_v1.24-amd64`

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-2.png?raw=true)

2. After clicking **Import**, select **Load Resource Package**, select the appropriate download source, and wait for the resource package download to complete.

    !!! note It is recommended that you choose Docker as the container engine for the K8s cluster. After selecting Docker as the container engine for K8s, Kuboard-Spray automatically uses Docker to run the various components of the K8s cluster, including containers on the Master node and the Worker node.

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-3.png?raw=true)

3. This `pulls` the relevant mirror dependencies:

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-4.png?raw=true)

4. After the mirrored resource pack is pulled successfully, return to Kuboard-Spray's web interface and see that the corresponding version of the resource pack has been imported.

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-5.png?raw=true)

#### Installing a Kubernetes Cluster

This chapter will guide you through the installation of the Kubernetes cluster.

1. Select **Cluster Management** and select **Add Cluster Installation Plan**:

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-6.png?raw=true)

2. In the pop-up dialog box, define the name of the cluster, select the version of the resource package you just imported, and click **OK**. As shown in the following figure:

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-7.png?raw=true)

##### Cluster planning

Kubernetes clusters are deployed in a pattern of `1 master + 1 worker +1 etcd` according to pre-defined role classifications.

After defining the completion cluster name in the previous step and selecting the completion resource pack version, click **OK** to proceed directly to the cluster planning phase.

1. Select the role and name of the corresponding node:

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-8.png?raw=true)

    - master node: Select the ETCD and control node and name it master0\. (You can also select the working node if you want the master node to work.) This approach improves resource utilization, but reduces the high availability of Kubernetes.)
    - worker node: Select only the worker node and name it node0.

2. After each node has filled in the role and node name, fill in the connection information for the corresponding node to the right, as shown in the following figure:

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-9.png?raw=true)

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-9-1.png?raw=true)

3. Click **Save** when you have filled out all the roles. Next you are ready to install the Kubernetes cluster.

#### Start installing Kubernetes cluster

After completing all roles in the previous step and **saving** them, click **Execute** to begin the installation of the Kubernetes cluster.

1. Click **OK** to begin installing the Kubernetes cluster as shown in the following figure:

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-10.png?raw=true)

2. When you install a Kubernetes cluster, the Kubernetes cluster is installed by executing an `ansible` script on the corresponding node. The overall event can take anywhere from 5 to 10 minutes depending on the machine configuration and network and the time to wait.

    __Note:__ If an error occurs, you can look at the contents of the log and confirm if it is a version mismatch for Kuboard-Spray. If it is, replace it with the appropriate version.

3. After installation, go to the master node of the Kubernetes cluster and execute `kubectl get node`:

    ```
    [root@master0 ~]# kubectl get node
    NAME      STATUS   ROLES                  AGE   VERSION
    master0   Ready    control-plane,master   52m   v1.23.17
    node0     Ready    <none>                 52m   v1.23.17
    ```

4. The command results are shown in the figure above, which indicates that the Kubernetes cluster installation is complete.

5. Adjust the DNS routing table on each node of Kubernetes. Execute the following command on each machine to locate the nameserver that contains `169.254.25.10` and delete the record. (The record may affect the efficiency of communication between the individual pods and need not be changed if it does not exist)

    ```
    vim /etc/resolve.conf 
    ```

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-10-1.png?raw=true)

## 2. Deployment helm

Helm is a package management tool for managing Kubernetes applications. It simplifies the process of deploying and managing applications by using charts (preconfigured installation package resources). Similar to APT for Ubuntu and YUM for CentOS, Helm provides a convenient way to install, upgrade, and manage Kubernetes applications.

Before installing Minio, we need to install Helm, as the Minio installation process depends on Helm. Here are the steps to install Helm:

__Note:__ This chapter operates at the master0 node.

1. Download the helm installation package:

    ```
    wget https://get.helm.sh/helm-v3.10.2-linux-amd64.tar.gz 
    #If in a restricted network environment in the country, you can change the following domestic mirror address 
    wget https://mirrors.huaweicloud.com/helm/v3.10.2/helm-v3.10.2-linux-amd64.tar.gz
    ```

2. Unzip and install:

    ```
    tar -zxf helm-v3.10.2-linux-amd64.tar.gz 
    mv linux-amd64/helm /usr/local/bin/helm 
    ```

3. Verify the version to see if the installation is complete:

    ```
    [root@k8s01 home]# helm version
    version.BuildInfo{Version:"v3.10.2", GitCommit:"50f003e5ee8704ec937a756c646870227d7c8b58", GitTreeState:"clean", GoVersion:"go1.18.8"}
    ```

    Installation is complete when the version information shown above appears.

## 3. CSI Deployment

CSI is a storage plug-in for Kubernetes and provides storage services for MinIO and MarixOne. This chapter will guide you through using the `local-path-provisioner` plugin.

__Note:__ This chapter operates at the master0 node.

1. Using the following command line, install CSI:

    ```
    wget https://github.com/rancher/local-path-provisioner/archive/refs/tags/v0.0.23.zip
    unzip v0.0.23.zip
    cd local-path-provisioner-0.0.23/deploy/chart/local-path-provisioner
    helm install --set nodePathMap[0].paths[0]="/opt/local-path-provisioner",nodePathMap[0].node=DEFAULT_PATH_FOR_NON_LISTED_NODES  --create-namespace --namespace local-path-storage local-path-storage ./
    ```

    If the original github address downloads too slowly, you can try downloading the mirror package from:

    ```
    wget https://githubfast.com/rancher/local-path-provisioner/archive/refs/tags/v0.0.23.zip 
    ```

2. After a successful installation, the command line appears as follows:

    ```
    root@master0:~# kubectl get pod -n local-path-storage
    NAME                                                        READY   STATUS    RESTARTS   AGE
    local-path-storage-local-path-provisioner-57bf67f7c-lcb88   1/1     Running   0          89s
    ```

    __Note:__ After installation, the storageClass provides storage services in the "/opt/local-path-provisioner" directory of the worker node. You can modify to another path.

3. Set the default `storageClass`:

    ```
    kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}' 
    ```

4. After setting the default successfully, the command line appears as follows:

    ```
    root@master0:~# kubectl get storageclass
    NAME                   PROVISIONER                                               RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
    local-path (default)   cluster.local/local-path-storage-local-path-provisioner   Delete          WaitForFirstConsumer   true                   115s
    ```

## 4. MinIO Deployment

 The role of MinIO is to provide object storage for MatrixOne. This chapter will guide you through deploying a single node MinIO.

__Note:__ This chapter operates at the master0 node.

### Installation Launch

1. The command line to install and start MinIO is as follows:

    ```
    helm repo add minio https://charts.min.io/ mkdir minio_ins && cd minio_ins helm fetch minio/minio ls -lth tar -zxvf minio-5.0.9.tgz # This version is subject to change, based on the actual download cd ./minio/

    kubectl create ns mostorage

    helm install minio \
    --namespace mostorage \
    --set resources.requests.memory=512Mi \
    --set replicas=1 \
    --set persistence.size=10G \
    --set mode=standalone \
    --set rootUser=rootuser,rootPassword=rootpass123 \
    --set consoleService.type=NodePort \
    --set image.repository=minio/minio \
    --set image.tag=latest \
    --set mcImage.repository=minio/mc \
    --set mcImage.tag=latest \
    -f values.yaml minio/minio 
    ```

    !!! note
        `--set resources.requests.memory=512Mi` sets MinIO's minimum memory consumption `--set persistence.size=1G` sets MinIO's storage size to 1G --`-set rootUser=rootuser,rootPassword=rootpass123` The parameters set here for rootUser and rootPassword are required for subsequent creation of the Kubernetes cluster's scrects file, so use a message that can be remembered. - If repeated multiple times for network or other reasons, you need to uninstall first:

             ```
             helm uninstall minio --namespace mostorage
             ```

2. After installing and starting MinIO successfully, the command line appears as follows:

    ```
    NAME: minio
    LAST DEPLOYED: Sun May  7 14:17:18 2023
    NAMESPACE: mostorage
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
    NOTES:
    MinIO can be accessed via port 9000 on the following DNS name from within your cluster:
    minio.mostorage.svc.cluster.local

    To access MinIO from localhost, run the below commands:

      1. export POD_NAME=$(kubectl get pods --namespace mostorage -l "release=minio" -o jsonpath="{.items[0].metadata.name}")

      2. kubectl port-forward $POD_NAME 9000 --namespace mostorage

    Read more about port forwarding here: http://kubernetes.io/docs/user-guide/kubectl/kubectl_port-forward/

    You can now access MinIO server on http://localhost:9000. Follow the below steps to connect to MinIO server with mc client:

      1. Download the MinIO mc client - https://min.io/docs/minio/linux/reference/minio-mc.html#quickstart

      2. export MC_HOST_minio-local=http://$(kubectl get secret --namespace mostorage minio -o jsonpath="{.data.rootUser}" | base64 --decode):$(kubectl get secret --namespace mostorage minio -o jsonpath="{.data.rootPassword}" | base64 --decode)@localhost:9000

      3. mc ls minio-local
    ```

    Minio has been successfully installed so far. During subsequent MatrixOne installations, MatrixOne will communicate with Minio directly through Kubernetes' Service (SVC) without additional configuration.

    However, if you want to connect to Minio from `localhost`, you can do the following command line to set the `POD_NAME` variable and connect `mostorage` to port 9000:

    ```
    export POD_NAME=$(kubectl get pods --namespace mostorage -l "release=minio" -o jsonpath="{.items[0].metadata.name}")
    nohup kubectl port-forward --address 0.0.0.0 $POD_NAME -n mostorage 9000:9000 &
    ```

3. Once launched, use <http://118.195.255.252:32001/> to log into MinIO's page and create the information stored by the object. As shown in the following figure, the account password is the rootUser and rootPassword set by `--set rootUser=rootuser,rootPassword=rootpass123` in the above steps:

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-13.png?raw=true)

4. Once the login is complete, you need to create an object to store the relevant information:

    Click **Bucket > Create Bucket** and fill in Bucket's name **minio-mo** in **Bucket Name**. Once completed, click the button **Create Bucket** at the bottom right.

    ![](https://github.com/matrixorigin/artwork/blob/main/docs/deploy/deploy-mo-cluster-14.png?raw=true)

## 5. MatrixOne Cluster Deployment

This chapter will guide you through deploying a MatrixOne cluster.

__Note:__ This chapter operates at the master0 node.

#### Installing MatrixOne-Operator

[MatrixOne Operator](https://github.com/matrixorigin/matrixone-operator) is a standalone software tool for deploying and managing MatrixOne clusters on Kubernetes. You can choose to deploy online or offline.

- **Deploy online**

Follow these steps to install MatrixOne Operator on master0. We will create a separate namespace `matrixone-operator` for Operator.

1. Add the matrixone-operator address to the helm repository:

    ```
    helm repo add matrixone-operator https://matrixorigin.github.io/matrixone-operator 
    ```

2. Update the repository:

    ```
    helm repo update 
    ```

3. View the MatrixOne Operator version:

    ```
    helm search repo matrixone-operator/matrixone-operator --versions --devel 
    ```

4. Specify the release to install MatrixOne Operator:

    ```
    helm install matrixone-operator matrixone-operator/matrixone-operator --version <VERSION> --create-namespace --namespace matrixone-operator 
    ```

    !!! note
        The parameter VERSION is the version number of the MatrixOne Operator to be deployed, such as 1.0.0-alpha.2.

5. After a successful installation, confirm the installation status using the following command:

    ```
    kubectl get pod -n matrixone-operator 
    ```

    Ensure that all Pod states in the above command output are Running.

    ```
    [root@master0 matrixone-operator]# kubectl get pod -n matrixone-operator
    NAME                                 READY   STATUS    RESTARTS   AGE
    matrixone-operator-f8496ff5c-fp6zm   1/1     Running   0          3m26s
    ```

The corresponding Pod states are normal as shown in the above code line.

- **Deploy offline**

You can select the Operator Release version installation package you need from the project's [Release list](https://github.com/matrixorigin/matrixone-operator/releases) for offline deployment.

1. Create a standalone namespace mo-op for Operator

    ```
    NS="mo-op" 
    kubectl create ns "${NS}" 
    kubectl get ns # return has mo-op 
    ```

2. Download and extract the matrixone-operator installation package

    ```
    wget https://github.com/matrixorigin/matrixone-operator/releases/download/chart-1.1.0-alpha2/matrixone-operator-1.1.0-alpha2.tgz
    tar xvf matrixone-operator-1.1.0-alpha2.tgz
    ```

    If the original github address downloads too slowly, you can try downloading the mirror package from:

    ```
    wget https://githubfast.com/matrixorigin/matrixone-operator/releases/download/chart-1.1.0-alpha2/matrixone-operator-1.1.0-alpha2.tgz
    ```
  
    After extraction it will be in the current directory production folder `matrixone-operator`.

3. Deploying matrixone-operator

    ```
    NS="mo-op" 
    cd matrixone-operator/ 
    helm install -n ${NS} mo-op ./charts/matrixone-operator --dependency-update # Success should return the status of deployed
    ```

    The above list of dependent docker mirrors is:

    - matrixone-operator
    - kruise-manager
  
    If you cannot pull a mirror from dockerhub, you can pull it from Aliyun using the following command:

    ```
    helm -n ${NS} install mo-op ./charts/matrixone-operator --dependency-update -set image.repository="registry.cn-hangzhou.aliyuncs.com/moc-pub/matrixone-operator" --set kruise.manager.image.repository="registry.cn-hangzhou.aliyuncs.com/moc-pub/kruise-manager"
    ```

    See matrixone-operator/values.yaml for details.

4. Check operator deployment status

    ```
    NS="mo-op" 
    helm list -n "${NS}" # returns a corresponding helm chart package with deployed 
    kubectl get pod -n "${NS}" -owide # returns a copy of a pod with Running
    ```

To learn more about Matrixone Operator, check out [Operator Administration](../../Deploy/MatrixOne-Operator-mgmt.md).

### Creating a MatrixOne Cluster

1. First create the namespace for MatrixOne:

    ```
    NS="mo-hn" 
    kubectl create ns ${NS} 
    ```

2. Customize the `yaml` file for the MatrixOne cluster by writing the following `mo.yaml` file:

    ```
    apiVersion: core.matrixorigin.io/v1alpha1
    kind: MatrixOneCluster
    metadata:
      name: mo
      namespace: mo-hn
    spec:
      # 1. Configuring tn
      tn:
        cacheVolume: # Disk Cache for tn
          size: 5Gi # Modified according to actual disk size and requirements
          storageClassName: local-path # If you don't write it, the default storage class will be used.
        resources:
          requests:
            cpu: 100m #1000m=1c
            memory: 500Mi # 1024Mi
          limits: # Note that the limits should not be lower than requests, nor exceed the capacity of a single node, and are generally allocated according to the actual situation, and it is sufficient to set the limits in line with requests.
            cpu: 200m
            memory: 1Gi
        config: |  # Configuration of tn
          [dn.Txn.Storage]
          backend = "TAE"
          log-backend = "logservice"
          [dn.Ckp]
          flush-interval = "60s"
          min-count = 100
          scan-interval = "5s"
          incremental-interval = "60s"
          global-interval = "100000s"
          [log]
          level = "error"
          format = "json"
          max-size = 512
        replicas: 1 # The number of copies of tn, which cannot be modified. The current version only supports a setting of 1.
      # 2. Configuring logservice
      logService:
        replicas: 3 # Number of copies of logservice
        resources:
          requests:
            cpu: 100m #1000m=1c
            memory: 500Mi # 1024Mi
          limits: # Note that the limits should not be lower than requests, nor exceed the capacity of a single node, and are generally allocated according to the actual situation, and it is sufficient to set the limits in line with requests.
            cpu: 200m
            memory: 1Gi
        sharedStorage: # Configuring s3 storage for logservice mapping
          s3:
            type: minio # The s3 storage type to which it is docked is minio
            path: minio-mo # The path to the minio bucket for mo, previously created via the console or the mc command.
            endpoint: http://minio.mostorage:9000 # Here is the svc address and port for the minio service
            secretRef: # Configure the key for accessing minio, i.e. secret, with the name minio
              name: minio
        pvcRetentionPolicy: Retain # Configure the cycle policy for pvc after cluster destruction, Retain for Retain and Delete for Delete.
        volume:
          size: 1Gi # Configure the size of the S3 object store, modify it according to the actual disk size and requirements
        config: | # Configuration of the logservice
          [log]
          level = "error"
          format = "json"
          max-size = 512
      # 3. 配置 cn
      tp:
        cacheVolume: # Disk Cache for cn
          size: 5Gi # Modified according to actual disk size and requirements
          storageClassName: local-path # If you don't write it, the default storage class will be used.
        resources:
          requests:
            cpu: 100m #1000m=1c
            memory: 500Mi # 1024Mi
          limits: # Note that the limits should not be lower than requests, nor exceed the capacity of a single node, and are generally allocated according to the actual situation, and it is sufficient to set the limits in line with requests.
            cpu: 200m
            memory: 2Gi
        serviceType: NodePort # The cn needs to provide an external access portal, and its svc is set to NodePort.
        nodePort: 31429 # nodePort Port Settings
        config: | # Configuring cn 
          [cn.Engine]
          type = "distributed-tae"
          [log]
          level = "debug"
          format = "json"
          max-size = 512
        replicas: 1
      version: nightly-54b5e8c # Here is the version of the MO image, you can check it through dockerhub, usually cn, tn, logservice are packaged in the same image, so you can use the same field to specify it, or you can specify it separately in the respective section, but without special circumstances, please use the unified image version.
      # https://hub.docker.com/r/matrixorigin/matrixone/tags
      imageRepository: matrixorigin/matrixone # Mirror repository address, if the tag has been modified since the local pull, then you can adjust this configuration item
      imagePullPolicy: IfNotPresent # Mirror pulling policy, consistent with official k8s configurable values
    ```

3. Create a Secret service in namespace `mo-hn` to access MinIO by executing the following command:

    ```
    kubectl -n mo-hn create secret generic minio --from-literal=AWS_ACCESS_KEY_ID=rootuser --from-literal=AWS_SECRET_ACCESS_KEY=rootpass123
    ```

    where the username and password use the `rootUser` and `rootPassword` set when creating the MinIO cluster.

4. Deploy the MatrixOne cluster by executing the following command:

    ```
    kubectl apply -f mo.yaml 
    ```

5. Please be patient for approximately 10 minutes, and continue if a Pod reboot occurs. until you see the following message indicating successful deployment:

    ```
    [root@master0 mo]# kubectl get pods -n mo-hn      
    NAME                                 READY   STATUS    RESTARTS      AGE
    mo-tn-0                              1/1     Running   0             74s
    mo-log-0                             1/1     Running   1 (25s ago)   2m2s
    mo-log-1                             1/1     Running   1 (24s ago)   2m2s
    mo-log-2                             1/1     Running   1 (22s ago)   2m2s
    mo-tp-cn-0                           1/1     Running   0             50s
    ```

### Dynamic expansion

The Operator supports dynamic expansion of the cacheVolume configuration of TN and CN, but does not support the reduction operation. There is no need to restart the CN and TN pods before and after the expansion process. The specific steps are as follows:

1. Make sure the StorageClass supports the volume extension feature

    ```bash
    #View storageclass name
    >kubectl get pvc -n ${MO_NS}

    #Check whether StorageClass supports volume extension function
    >kubectl get storageclass ${SC_NAME} -oyaml |  grep allowVolumeExpansion #SC_NAME is the sc type of pvc used by the mo cluster
    allowVolumeExpansion: true #Only when true can subsequent steps be performed
    ```

2. Enter cluster configuration editing mode

    ```bash
    kubectl edit mo -n ${MO_NS} ${MO_NAME} # Among them, MO_NS is the namespace where the MO cluster is deployed, and MO_NAME is the name of the MO cluster; for example,MO_NS=matrixone; MO_NAME=mo_cluster
    ```

3. Modify the cacheVolume size of tn and cn as needed

    ```bash
    - cacheVolume:
            size: 900Gi
    ```

    - If it is a CN group, modify spec.cnGroups[0].cacheVolume (or spec.cnGroups[1].cacheVolume, where n in [n] is the array subscript of the CN Group);
    - If it is CN, modify spec.tp.cacheVolume;
    - If TN, modifies spec.tn.cacheVolume (or spec.dn.cacheVolume).

    Then save and exit: press `esq` keys, and `:wq`

4. View expansion results

    ```bash
    #The value of the `capacity` field is the value after expansion.
    >kubectl get pvc -n ${MO_NS}

    >kubectl get pv | grep ${NS}
    ```

## Connecting a MatrixOne Cluster

In order to connect to a MatrixOne cluster, you need to map the port of the corresponding service to the MatrixOne node. Here is a guide for connecting to a MatrixOne cluster using `kubectl port-forward`:

- Allow local access only:

   ```
   nohup kubectl port-forward -nmo-hn svc/mo-tp-cn 6001:6001 & 
   ```

- Specify that a machine or all machines access:

   ```
   nohup kubectl port-forward -nmo-hn --address 0.0.0.0 svc/mo-tp-cn 6001:6001 & 
   ```

After specifying **allow local access** or **specifying a machine** or all, you can connect to MatrixOne using a MySQL client:

```
# Connect to MySQL service using 'mysql' command line tool
# Use 'kubectl get svc/mo-tp-cn -n mo-hn -o jsonpath='{.spec.clusterIP}' to get the cluster IP address of the service in the Kubernetes cluster
# The '-h' parameter specifies the hostname or IP address of the MySQL service
# The '-P' parameter specifies the port number of the MySQL service, in this case 6001
# '-uroot' means login as root
# '-p111' indicates that the initial password is 111
mysql -h $(kubectl get svc/mo-tp-cn -n mo-hn -o jsonpath='{.spec.clusterIP}') -P 6001 -uroot -p111
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 163
Server version: 8.0.30-MatrixOne-v1.1.1 MatrixOne

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

After displaying `mysql>`, the distributed MatrixOne cluster setup connection completes.

!!! note
    The login account in the above code section is the initial account. Please change the initial password promptly after logging into MatrixOne, see [Password Management](../../Security/password-mgmt.md).
