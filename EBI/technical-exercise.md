# Kubernetes Cluster Deployment and Management with RKE2

<!-- TOC -->
* [Kubernetes Cluster Deployment and Management with RKE2](#kubernetes-cluster-deployment-and-management-with-rke2)
  * [Access Details](#access-details)
  * [1. Cluster Setup](#1-cluster-setup)
    * [Gather Node Details](#gather-node-details)  
    * [Step 1: Install Rancher on Bastion](#step-1-install-rancher-on-bastion)
    * [Step 2: Configure RKE2 Cluster](#step-2-configure-rke2-cluster)
    * [Step 3: Bootstrap Master Node](#step-3-bootstrap-master-node)
    * [Step 4: Bootstrap Worker Node](#step-4-bootstrap-worker-node)
    * [Step 5: Deploy Essential Addons](#step-5-deploy-essential-addons)
      * [Monitoring](#monitoring)
      * [Persistent Volume](#persistent-volume)
  * [2. Provisioning WordPress](#2-provisioning-wordpress)
    * [Step 1: Perform a initial deployment](#step-1-perform-a-initial-deployment)
    * [Step 2: Scale WordPress](#step-2-scale-wordpress)
      * [Error 1: PV mounting issue](#error-1-pv-mounting-issue)
    * [Step 3: Access WordPress and Perform Initial Setup](#step-3-access-wordpress-and-perform-initial-setup)
  * [3. Upgrading the cluster](#3-upgrading-the-cluster)
    * [Step 1. Take Snapshots](#step-1-take-snapshots)
    * [Step 2. Set up external monitoring](#step-2-set-up-external-monitoring)
    * [Step 3. Perform the upgrade](#step-3-perform-the-upgrade)
  * [4. Achieving Near-Zero Downtime](#4-achieving-near-zero-downtime)
    * [Step 1. Deploy MariaDB Galera Cluster](#step-1-deploy-mariadb-galera-cluster)
    * [Step 2. Create a database and user](#step-2-create-a-database-and-user)
    * [Step 3. Update WordPress Helm Chart](#step-3-update-wordpress-helm-chart)
  * [Conclusion](#conclusion)
    * [References:](#references)
<!-- TOC -->

## Access Details

| Component                                 | Details                                                                                                                                                    |
|-------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Rancher                                   | URL: [`https://45.88.80.253`]()                                                                                                                            |
|                                           | Username: `admin`                                                                                                                                          |
|                                           | Password: `qodweg-semte2-Cibtyd`                                                                                                                           |
| Kubernetes Cluster Name                   | Demo                                                                                                                                                       |
| Kubernetes Dashboard                      | [`https://45.88.80.253/dashboard/c/c-m-4nr6hwr7/explorer#cluster-events`]()                                                                                |
| Helm Releases for wordpress deployment    | [`https://45.88.80.253/dashboard/c/c-m-4nr6hwr7/apps/catalog.cattle.io.app`]()                                                                             |
| Reverse Proxy URL for external monitoring | [`http://wordpress.45.88.80.253.sslip.io:8080`]()                                                                                                          |
| Uptime Robot Status Page                  | [`https://stats.uptimerobot.com/Y1wd118u9X`]()                                                                                                             |
| Grafana Dashboards                        | [`https://45.88.80.253/k8s/clusters/c-m-4nr6hwr7/api/v1/namespaces/cattle-monitoring-system/services/http:rancher-monitoring-grafana:80/proxy/?orgId=1`]() | 


## 1. Cluster Setup

### Gather Node Details

```shell
$ cat ~/.ssh/config
host rke-bastion
  hostname 45.88.80.253
  identityfile ~/.ssh/ebi-exercise/key.pem
  user ubuntu

host rke-master
  hostname 192.168.10.129
  identityfile ~/.ssh/ebi-exercise/key.pem
  user ubuntu
  proxyjump rke-bastion

host rke-worker
  hostname 192.168.10.97
  identityfile ~/.ssh/ebi-exercise/key.pem
  user ubuntu
  proxyjump rke-bastion
```

Let's start by assessing the capacity of the machines so that we can plan the cluster accordingly.

```shell
$ lscpu | grep '^CPU(s):'
CPU(s):                             4

$ cat /proc/meminfo | grep 'MemTotal'
MemTotal:        8128004 kB

$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           794M  1.2M  793M   1% /run
/dev/vda1        78G  2.0G   76G   3% /
tmpfs           3.9G     0  3.9G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
/dev/vda15      105M  6.1M   99M   6% /boot/efi
tmpfs           794M  4.0K  794M   1% /run/user/1000
```
###### Node Details

| Node      | Role    | IP             | CPU (number) | Memory (GB) | Storage (GB) |
|-----------|---------|----------------|--------------|-------------|--------------|
| bastion-1 | bastion | 45.88.80.253   | 4            | 8           | 80           |
| master-1  | master  | 192.168.10.129 | 4            | 8           | 80           |
| worker-1  | worker  | 192.168.10.97  | 4            | 8           | 80           |


### Step 1: Install Rancher on Bastion

We will start by configuring Rancher on the bastion node. Rancher is a Kubernetes management platform that allows us to 
manage and deploy Kubernetes clusters.

#### Install Prerequisites

Install docker.
```shell
$ sudo apt install docker.io
```

Install kubectl and helm.
```shell
$ sudo snap install kubectl --classic
helm 1.29.3 from Snapcrafters✪ installed
$ sudo snap install helm --classic
helm 3.14.3 from Snapcrafters✪ installed
```

Bring up the Rancher container.
```shell
$ docker run --privileged -d --name rancher --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher:v2.8.3-rc6
4fd5bff9f44243487146f4bb7e973d8abb27a92d8a18b8fcef1f87a2284cbb0a
```

Get the bootstrap password for initializing Rancher.
```shell
$ docker logs rancher 2>&1 | grep "Bootstrap Password:"
2024/03/24 23:52:17 [INFO] Bootstrap Password: r5hf9bxhh66p4fwqdj974mnbqwnlqvlk8p6tfhgx5v2hbfl5tt5587
```

### Step 2: Configure RKE2 Cluster

Lets access the Rancher UI from our browser using the IP address of the bastion node [`https://45.88.80.253`](). We will
use the bootstrap password to initialize Rancher.

Once logged in, we will create a new cluste r and select the RKE2 option. Navigate to the `Cluster Management` tab from 
the left pane, and then to `Clusters` section. On the top right corner, click on `Create`. Select the `RKE2` option and
provide the cluster name as `Demo`.

For now, we will create a v1.27 cluster so that we can upgrade it to v1.28 after we deploy wordpress.
![Screenshot 2024-03-26 at 15.47.50.png](img%2FScreenshot%202024-03-26%20at%2015.47.50.png)

In order for the applications to not go down during cluster maintenance it is important we configure the cluster with
proper node draining configuration. We will set the `Update Strategy` to the following values:
![Screenshot 2024-03-26 at 16.00.10.png](img%2FScreenshot%202024-03-26%20at%2016.00.10.png)

Let's proceed with the cluster creation.

### Step 3: Bootstrap Master Node

Once we create the `Demo` cluster, we will be provided with the `rke2` command to bootstrap the nodes. Using the
provided command, we will bootstrap the master node.

```shell
$ curl --insecure -fL https://45.88.80.253/system-agent-install.sh | sudo  sh -s - --server https://45.88.80.253 --label 'cattle.io/os=linux' --token x6s8w9st22jf6j9lrcbgrn2q528j5f962s6flkzdk5j9gl9gqnz2b8 --ca-checksum c8019bafeade0741fd36e31373d6cb4d43cc62146bff4df661bfb140bb8bf998 --etcd --controlplane --worker
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 32283    0 32283    0     0  1929k      0 --:--:-- --:--:-- --:--:-- 1970k
[INFO]  Label: cattle.io/os=linux
[INFO]  Role requested: etcd
[INFO]  Role requested: controlplane
[INFO]  Role requested: worker
[INFO]  Using default agent configuration directory /etc/rancher/agent
[INFO]  Using default agent var directory /var/lib/rancher/agent
[INFO]  Determined CA is necessary to connect to Rancher
[INFO]  Successfully downloaded CA certificate
[INFO]  Value from https://45.88.80.253/cacerts is an x509 certificate
[INFO]  Successfully tested Rancher connection
[INFO]  Downloading rancher-system-agent binary from https://45.88.80.253/assets/rancher-system-agent-amd64
[INFO]  Successfully downloaded the rancher-system-agent binary.
[INFO]  Downloading rancher-system-agent-uninstall.sh script from https://45.88.80.253/assets/system-agent-uninstall.sh
[INFO]  Successfully downloaded the rancher-system-agent-uninstall.sh script.
[INFO]  Generating Cattle ID
[INFO]  Successfully downloaded Rancher connection information
[INFO]  systemd: Creating service file
[INFO]  Creating environment file /etc/systemd/system/rancher-system-agent.env
[INFO]  Enabling rancher-system-agent.service
[INFO]  Starting/restarting rancher-system-agent.service
```

### Step 4: Bootstrap Worker Node

Using the same command, we will bootstrap the worker node, except this time we will not pass `--etcd` and 
`--controlplane` flags.

```shell
curl --insecure -fL https://45.88.80.253/system-agent-install.sh | sudo  sh -s - --server https://45.88.80.253 --label 'cattle.io/os=linux' --token x6s8w9st22jf6j9lrcbgrn2q528j5f962s6flkzdk5j9gl9gqnz2b8 --ca-checksum c8019bafeade0741fd36e31373d6cb4d43cc62146bff4df661bfb140bb8bf998 --worker
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 32283    0 32283    0     0  2284k      0 --:--:-- --:--:-- --:--:-- 2425k
[INFO]  Label: cattle.io/os=linux
[INFO]  Role requested: worker
[INFO]  Using default agent configuration directory /etc/rancher/agent
[INFO]  Using default agent var directory /var/lib/rancher/agent
[INFO]  Determined CA is necessary to connect to Rancher
[INFO]  Successfully downloaded CA certificate
[INFO]  Value from https://45.88.80.253/cacerts is an x509 certificate
[INFO]  Successfully tested Rancher connection
[INFO]  Downloading rancher-system-agent binary from https://45.88.80.253/assets/rancher-system-agent-amd64
[INFO]  Successfully downloaded the rancher-system-agent binary.
[INFO]  Downloading rancher-system-agent-uninstall.sh script from https://45.88.80.253/assets/system-agent-uninstall.sh
[INFO]  Successfully downloaded the rancher-system-agent-uninstall.sh script.
[INFO]  Generating Cattle ID
[INFO]  Successfully downloaded Rancher connection information
[INFO]  systemd: Creating service file
[INFO]  Creating environment file /etc/systemd/system/rancher-system-agent.env
[INFO]  Enabling rancher-system-agent.service
Created symlink /etc/systemd/system/multi-user.target.wants/rancher-system-agent.service → /etc/systemd/system/rancher-system-agent.service.
[INFO]  Starting/restarting rancher-system-agent.service
```

Once the nodes are bootstrapped, we will see them in the Rancher UI under the `Nodes` section.

Kubernetes Config file is placed under `/home/ubuntu/.kube/config` in the bastion node.

### Step 5: Deploy Essential Addons

#### Monitoring

Let's add prometheus and grafana monitoring to the cluster. We will use the Rancher monitoring helm chart to deploy the
monitoring stack. We can easily install the monitoring stack by navigating to the `Cluster Tools` button and then to the
clicking on the `Install` button for the `Monitoring` tool. Reduce the CPU and memory to 250m and 250Mi respectively,
considering the size of the nodes.

Rancher will execute the following command to install the monitoring stack.
```shell
helm upgrade --install=true --namespace=cattle-monitoring-system --timeout=10m0s --values=/home/shell/helm/values-rancher-monitoring-crd-103.0.4-up45.31.1.yaml --version=103.0.4+up45.31.1 --wait=true rancher-monitoring-crd /home/shell/helm/rancher-monitoring-crd-103.0.4-up45.31.1.tgz
```

#### Persistent Volume
In order to automate the lifecycle of the persistent volumes, we will deploy the Longhorn storage system. We will use 
the Longhorn helm chart to deploy the storage system. Longhorn is available under the same `Cluster Tools` section.

Rancher will execute the following command to install Longhorn.
```shell
helm upgrade --install=true --namespace=longhorn-system --timeout=10m0s --values=/home/shell/helm/values-longhorn-crd-103.2.2-up1.5.4.yaml --version=103.2.2+up1.5.4 --wait=true longhorn-crd /home/shell/helm/longhorn-crd-103.2.2-up1.5.4.tgz
2024-03-25T01:00:36.701468690Z Release "longhorn-crd" does not exist. Installing it now.
```

Once advantage of Longhorn is that it automatic replication of the volumes for high availability. The default replicas
are three, but we will reduce it to two. It also provides a storage class for dynamic provisioning of the volumes which
makes life easier for the application deployment later on. Longhorn takes care of the backup and restore of the volumes
in case of a disaster.


## 2. Provisioning WordPress

### Step 1: Perform a initial deployment

We will use bitnami WordPress helm chart for deploying. Let's start by inspecting the WordPress helm chart. 

```shell
helm template my-release oci://registry-1.docker.io/bitnamicharts/wordpress
```

###### What we are looking for?
1. What resources are being created?
2. Is there a provision to scale the deployment?
3. How does the chart handles liveness and readiness probes?
4. Is there a provision to specify the storage class for the PVCs?
5. What are the default values for the chart?

After reading the documentation, the following values are set in the Rancher UI under `Apps` -> `Charts` -> `WordPress`
Chart.

```yaml
mariadb:
  enabled: true
  primary:
    persistence:
      accessModes:
        - ReadWriteOnce
      enabled: true
      size: 1Gi
      storageClass: 'longhorn'
persistence:
  accessModes:
    - ReadWriteOnce
  size: 1Gi
  storageClass: 'longhorn'
service:
  type: NodePort
wordpressBlogName: Surendhar's Blog!
wordpressEmail: suren@ebi.ac.uk
wordpressFirstName: Surendhar
wordpressLastName: Ravichandran
wordpressSkipInstall: true
wordpressUsername: suren
```

This configuration provisions a Mariadb deployment in standalone mode and deploys wordpress.Rancher used the following 
command to install the WordPress helm chart.
```shell
helm install --namespace=wordpress --timeout=10m0s --values=/home/shell/helm/values-wordpress-20.1.2.yaml --version=20.1.2 --wait=true wordpress /home/shell/helm/wordpress-20.1.2.tgz
```

### Step 2: Scale WordPress

In order for the wordpress deployment to be highly available, we will scale the deployment to two replicas. We will add
the following configurations to the helm chart and upgrade the deployment.

```yaml
replicaCount: 2
persistence:
  accessModes:
    - ReadWriteMany
```

#### Error 1: PV mounting issue
When deploying the helm chart, the deployment was stuck. Upon inspecting the deployment, it was found that pods could
not mount the volumes provided by Longhorn.

```shell
$ kubectl -n wordpress describe deployment wordpress
...
Events:
  Type     Reason                  Age              From                     Message
  ----     ------                  ----             ----                     -------
  Warning  FailedScheduling        54s              default-scheduler        0/2 nodes are available: pod has unbound immediate PersistentVolumeClaims. preemption: 0/2 nodes are available: 2 Preemption is not helpful for scheduling..
  Warning  FailedScheduling        53s              default-scheduler        0/2 nodes are available: pod has unbound immediate PersistentVolumeClaims. preemption: 0/2 nodes are available: 2 Preemption is not helpful for scheduling..
  Normal   Scheduled               51s              default-scheduler        Successfully assigned wordpress/wordpress-65cb4497bf-49chv to master-1
  Normal   SuccessfulAttachVolume  9s               attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-324c6fcb-7503-4193-9d87-47632c55a621"
  Warning  FailedMount             0s (x5 over 8s)  kubelet                  MountVolume.MountDevice failed for volume "pvc-324c6fcb-7503-4193-9d87-47632c55a621" : rpc error: code = Internal desc = mount failed: exit status 32
Mounting command: /usr/local/sbin/nsmounter
Mounting arguments: mount -t nfs -o vers=4.1,noresvport,timeo=600,retrans=5,softerr 10.43.155.15:/pvc-324c6fcb-7503-4193-9d87-47632c55a621 /var/lib/kubelet/plugins/kubernetes.io/csi/driver.longhorn.io/40283ba00d1768304931f96955025a27ce7f2f116a3a10e6bae83b1a443486a3/globalmount
Output: mount: /var/lib/kubelet/plugins/kubernetes.io/csi/driver.longhorn.io/40283ba00d1768304931f96955025a27ce7f2f116a3a10e6bae83b1a443486a3/globalmount: bad option; for several filesystems (e.g. nfs, cifs) you might need a /sbin/mount.<type> helper program.
```
Upon checking the longhorn documentation, For Read write many volumes, longhorn uses NFS server to provision them. 
We need to install nfs package in all the nodes.

```shell
$ sudo apt-get install nfs-common
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  keyutils libnfsidmap1 rpcbind
Suggested packages:
  watchdog
The following NEW packages will be installed:
  keyutils libnfsidmap1 nfs-common rpcbind
0 upgraded, 4 newly installed, 0 to remove and 0 not upgraded.
Need to get 381 kB of archives.
After this operation, 1447 kB of additional disk space will be used.
Do you want to continue? [Y/n] y
Get:1 http://nova.clouds.archive.ubuntu.com/ubuntu jammy-updates/main amd64 libnfsidmap1 amd64 1:2.6.1-1ubuntu1.2 [42.9 kB]
Get:2 http://nova.clouds.archive.ubuntu.com/ubuntu jammy/main amd64 rpcbind amd64 1.2.6-2build1 [46.6 kB]
Get:3 http://nova.clouds.archive.ubuntu.com/ubuntu jammy/main amd64 keyutils amd64 1.6.1-2ubuntu3 [50.4 kB]
Get:4 http://nova.clouds.archive.ubuntu.com/ubuntu jammy-updates/main amd64 nfs-common amd64 1:2.6.1-1ubuntu1.2 [241 kB]
Fetched 381 kB in 0s (6250 kB/s)
Selecting previously unselected package libnfsidmap1:amd64.
(Reading database ... 93943 files and directories currently installed.)
Preparing to unpack .../libnfsidmap1_1%3a2.6.1-1ubuntu1.2_amd64.deb ...
Unpacking libnfsidmap1:amd64 (1:2.6.1-1ubuntu1.2) ...
Selecting previously unselected package rpcbind.
Preparing to unpack .../rpcbind_1.2.6-2build1_amd64.deb ...
Unpacking rpcbind (1.2.6-2build1) ...
Selecting previously unselected package keyutils.
Preparing to unpack .../keyutils_1.6.1-2ubuntu3_amd64.deb ...
Unpacking keyutils (1.6.1-2ubuntu3) ...
Selecting previously unselected package nfs-common.
Preparing to unpack .../nfs-common_1%3a2.6.1-1ubuntu1.2_amd64.deb ...
Unpacking nfs-common (1:2.6.1-1ubuntu1.2) ...
Setting up libnfsidmap1:amd64 (1:2.6.1-1ubuntu1.2) ...
Setting up rpcbind (1.2.6-2build1) ...
Created symlink /etc/systemd/system/multi-user.target.wants/rpcbind.service → /lib/systemd/system/rpcbind.service.
Created symlink /etc/systemd/system/sockets.target.wants/rpcbind.socket → /lib/systemd/system/rpcbind.socket.
Setting up keyutils (1.6.1-2ubuntu3) ...
Setting up nfs-common (1:2.6.1-1ubuntu1.2) ...

Creating config file /etc/idmapd.conf with new version

Creating config file /etc/nfs.conf with new version
Adding system user `statd' (UID 114) ...
Adding new user `statd' (UID 114) with group `nogroup' ...
Not creating home directory `/var/lib/nfs'.
Created symlink /etc/systemd/system/multi-user.target.wants/nfs-client.target → /lib/systemd/system/nfs-client.target.
Created symlink /etc/systemd/system/remote-fs.target.wants/nfs-client.target → /lib/systemd/system/nfs-client.target.
auth-rpcgss-module.service is a disabled or a static unit, not starting it.
nfs-idmapd.service is a disabled or a static unit, not starting it.
nfs-utils.service is a disabled or a static unit, not starting it.
proc-fs-nfsd.mount is a disabled or a static unit, not starting it.
rpc-gssd.service is a disabled or a static unit, not starting it.
rpc-statd-notify.service is a disabled or a static unit, not starting it.
rpc-statd.service is a disabled or a static unit, not starting it.
rpc-svcgssd.service is a disabled or a static unit, not starting it.
rpc_pipefs.target is a disabled or a static unit, not starting it.
var-lib-nfs-rpc_pipefs.mount is a disabled or a static unit, not starting it.
Processing triggers for man-db (2.10.2-1) ...
Processing triggers for libc-bin (2.35-0ubuntu3.6) ...
Scanning processes...
Scanning candidates...
Scanning linux images...

Restarting services...
 systemctl restart rke2-server.service
Service restarts being deferred:                                                          /etc/needrestart/restart.d/dbus.service                                                  systemctl restart networkd-dispatcher.service
 systemctl restart unattended-upgrades.service

No containers need to be restarted.

No user sessions are running outdated binaries.

No VM guests are running outdated hypervisor (qemu) binaries on this host.
```

As soon as the NFS client was installed, the PVC was successfully mounted and deployment was successful.

```shell
helm upgrade --history-max=5 --install=true --namespace=wordpress --timeout=10m0s --values=/home/shell/helm/values-wordpress-20.1.2.yaml --version=20.1.2 --wait=true wordpress /home/shell/helm/wordpress-20.1.2.tgz
checking 11 resources for changes
Patch NetworkPolicy "wordpress-mariadb" in namespace wordpress
Looks like there are no changes for ServiceAccount "wordpress-mariadb"
Looks like there are no changes for ServiceAccount "wordpress"
Looks like there are no changes for Secret "wordpress-mariadb"
Looks like there are no changes for Secret "wordpress"
Looks like there are no changes for ConfigMap "wordpress-mariadb"
Looks like there are no changes for PersistentVolumeClaim "wordpress"
Patch Service "wordpress-mariadb" in namespace wordpress
Looks like there are no changes for Service "wordpress"
Patch Deployment "wordpress" in namespace wordpress
Patch StatefulSet "wordpress-mariadb" in namespace wordpress
beginning wait for 11 resources with timeout of 10m0s
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
...
StatefulSet is ready: wordpress/wordpress-mariadb. 1 out of 1 expected pods are ready
Release "wordpress" has been upgraded. Happy Helming!
NAME: wordpress
LAST DEPLOYED: Mon Mar 25 21:09:05 2024
NAMESPACE: wordpress
STATUS: deployed
REVISION: 2
TEST SUITE: None
NOTES:
CHART NAME: wordpress
CHART VERSION: 20.1.2
APP VERSION: 6.4.3

** Please be patient while the chart is being deployed **

Your WordPress site can be accessed through the following DNS name from within your cluster:

    wordpress.wordpress.svc.cluster.local (port 80)

To access your WordPress site from outside the cluster follow the steps below:

1. Get the WordPress URL by running these commands:

   export NODE_PORT=$(kubectl get --namespace wordpress -o jsonpath="{.spec.ports[0].nodePort}" services wordpress)
   export NODE_IP=$(kubectl get nodes --namespace wordpress -o jsonpath="{.items[0].status.addresses[0].address}")
   echo "WordPress URL: http://$NODE_IP:$NODE_PORT/"
   echo "WordPress Admin URL: http://$NODE_IP:$NODE_PORT/admin"

2. Open a browser and access WordPress using the obtained URL.

3. Login with the following credentials below to see your blog:

  echo Username: suren
  echo Password: $(kubectl get secret --namespace wordpress wordpress -o jsonpath="{.data.wordpress-password}" | base64 -d)

WARNING: There are "resources" sections in the chart not set. Using "resourcesPreset" is not recommended for production. For production installations, please set the following values according to your workload needs:
  - resources
+info https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/

---------------------------------------------------------------------
SUCCESS: helm upgrade --history-max=5 --install=true --namespace=wordpress --timeout=10m0s --values=/home/shell/helm/values-wordpress-20.1.2.yaml --version=20.1.2 --wait=true wordpress /home/shell/helm/wordpress-20.1.2.tgz
---------------------------------------------------------------------
```

### Step 3: Access WordPress and Perform Initial Setup

Login to the Rancher UI, from the `demo` cluster page, download the kubernetes config file using `Download KubeConfig`
icon in the top right corner. There is also a direct link in the top section of this page. You can perform the steps
from you local machines as Rancher provides a proxy for the Kube API Server and the kubernetes config file is pointing
to the proxy URL.

Once downloaded set the KUBECONFIG environment variable to the downloaded file and create a portforwarding to the
wordpress service.

```shell
export KUBECONFIG=/Users/suren/Downloads/demo.yaml
kubectl -n wordpress port-forward service/wordpress 8080:80
```

Now we can access the WordPress site by navigating to [`http://localhost:8080`](). After completing the inital setup
we land to the welcome page.

![Screenshot 2024-03-25 at 22.03.48.png](img%2FScreenshot%202024-03-25%20at%2022.03.48.png)
![Screenshot 2024-03-25 at 22.05.10.png](img%2FScreenshot%202024-03-25%20at%2022.05.10.png)
![Screenshot 2024-03-25 at 22.05.23.png](img%2FScreenshot%202024-03-25%20at%2022.05.23.png)


## 3. Upgrading the cluster

### Step 1. Take Snapshots
Before upgrading the cluster, let's take the snapshot of the cluster. From the Rancher UI Cluster management tab, Under 
cluster `demo`, we can take snapshot of the cluster.
![Screenshot 2024-03-26 at 18.17.04.png](img%2FScreenshot%202024-03-26%20at%2018.17.04.png)
Once the snapshots are takes, they can be viewed under the `Snapshots` section under the same page.

### Step 2. Set up external monitoring
In order to view the availability of the WordPress blog, let's use [UptimeRobot](https://uptimerobot.com). Since our
nodes are in the private subnet and accessible only from the bastion node, let's enable ingress on the kubernetes
cluster and configure a nginx service as a reverse proxy in the bastion node.

Let's modify the WordPress helm chart to include the following values.

```yaml
ingress:
  enabled: true
  hostname: wordpress.45.88.80.253.sslip.io
  ingressClassName: 'nginx'
  path: /
  pathType: ImplementationSpecific
mariadb:
  enabled: true
  primary:
    persistence:
      accessModes:
        - ReadWriteOnce
      enabled: true
      size: 1Gi
      storageClass: 'longhorn'
persistence:
  accessModes:
    - ReadWriteMany
  size: 1Gi
  storageClass: 'longhorn'
replicaCount: 2
service:
  type: NodePort
wordpressBlogName: Surendhar's Blog!
wordpressEmail: suren@ebi.ac.uk
wordpressFirstName: Surendhar
wordpressLastName: Ravichandran
wordpressSkipInstall: true
wordpressUsername: suren
```

On the bastion node, we will deploy the nginx service as a reverse proxy to the nginx ingress service in the kubernetes.

```shell
$ sudo apt install nginx
````

Create a new configuration file in the `/etc/nginx/sites-available/ingress` with the following content.

```nginx
  upstream backend {
    # Round robin balancing between two servers
    server 192.168.10.129:80 weight=1;
    server 192.168.10.97:80 weight=1;
  }

  server {
    listen 8080 default_server;

    # Forward all requests to the backend servers
    location / {
      proxy_pass http://backend;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_set_header X-Forwarded-Host $host;
      proxy_set_header X-Forwarded-Port $server_port;
    }
  }
 ```
Test nginx configuration.
```shell
$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Enable the site and restart the nginx service.

```shell
$ sudo ln -s /etc/nginx/sites-available/ingress /etc/nginx/sites-enabled/ingress
$ sudo systemctl reload nginx
```

Now we can access the WordPress site from the external world using the URL 
[`http://wordpress.45.88.80.253.sslip.io:8080`]().

Note: The images not loading in the WordPress site when accessed from the nginx reverse proxy. For monitoring the
availability of the WordPress site, this is good enough. The Uptime Robot service depends upon the HTTP status code 
returned by the site.

Now, just add the external URL in Uptime Robot and we are good to monitor the availability of the WordPress site.

![Screenshot 2024-03-25 at 23.13.27.png](img%2FScreenshot%202024-03-25%20at%2023.13.27.png)

### Step 3. Perform the upgrade

From the Rancher UI, navigate to the `Cluster Management` tab and then to the `Clusters` section, click on the details
button for the `demo` cluster. In the pop-up menu, select, `Edit Config`, under `Basics`, we can change the Kubernetes
version to `v1.28.0`. Click on `Save` to save the changes.
![Screenshot 2024-03-25 at 23.34.05.png](img%2FScreenshot%202024-03-25%20at%2023.34.05.png)

Now Rancher will start the upgrade process. The upgrade process can be monitored from the `demo` cluster page. Once the
upgrade is completed, the cluster will be running on the new version.

![Screenshot 2024-03-25 at 23.36.32.png](img%2FScreenshot%202024-03-25%20at%2023.36.32.png)

Verify the kubectl version by running the following command.
```shell
$ kubectl get nodes -o wide
NAME       STATUS   ROLES                              AGE   VERSION          INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
master-1   Ready    control-plane,etcd,master,worker   42h   v1.28.7+rke2r1   192.168.10.129   <none>        Ubuntu 22.04.4 LTS   5.15.0-100-generic   containerd://1.7.11-k3s2
worker-1   Ready    worker                             42h   v1.28.7+rke2r1   192.168.10.97    <none>        Ubuntu 22.04.4 LTS   5.15.0-100-generic   containerd://1.7.11-k3s2
```

## 4. Achieving Near-Zero Downtime

Upon closely watching the upgrade process, it took around 3 minutes for the mariadb pod to start up, even though its 
persistence volume is readily available on the other node. The WordPress pods were waiting for the database connection 
to be ready. This cause the WordPress service to be down for around 5 minutes during the cluster upgrade.

Clearly, having one mariadb instance is a single point of failure. Let's fix this by moving to a mariadb galera cluster.

### Step 1. Deploy MariaDB Galera Cluster

Bitnami provides a helm chart for deploying MariaDB Galera Cluster. Let's inspect the helm chart to understand the
resources created and the configurations available.

```shell
helm template my-release oci://registry-1.docker.io/bitnami/mariadb-galera
```

After checking the documentation of the helm chart, we can provision the MariaDB Galera Cluster with the following 
values.

```yaml
global:
  storageClass: longhorn
persistence:
  size: 1Gi
replicaCount: 2
```

Deploy the MariaDB Galera Cluster using the following command.

```shell
helm upgrade --install db --namespace wordpress oci://registry-1.docker.io/bitnamicharts/mariadb-galera -f values.yaml
```

### Step 2. Create a database and user

Fetch the root password for the MariaDB Galera Cluster using the following command.
```shell
kubectl get secret --namespace wordpress db-mariadb-galera -o jsonpath="{.data.mariadb-root-password}" | base64 -d
```
Login to the MariaDB Galera Cluster using the following command.
```shell
$ kubectl exec -it db-mariadb-galera-0 -- /bin/bash
$ mariadb -h db-mariadb-galera.wordpress.svc -u root -p
```
Create a database and user for the WordPress site.
```mariadb
CREATE DATABASE wordpress CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
GRANT ALL PRIVILEGES ON wordpress.* TO 'wp_user'@'%' IDENTIFIED BY 'wp_password';
FLUSH PRIVILEGES;
EXIT;
```

### Step 3. Update WordPress Helm Chart

Update the WordPress helm chart to use the MariaDB Galera Cluster. The following values are set in the Rancher UI under
`Apps` -> `Charts` -> `WordPress` Chart.

```yaml
externalDatabase:
  database: wordpress
  host: db-mariadb-galera.wordpress.svc
  password: wp_password
  port: 3306
  user: wp_user
ingress:
  enabled: true
  hostname: wordpress.45.88.80.253.sslip.io
  ingressClassName: 'nginx'
  path: /
  pathType: ImplementationSpecific
persistence:
  accessModes:
    - ReadWriteMany
  size: 1Gi
  storageClass: 'longhorn'
replicaCount: 2
service:
  type: NodePort
wordpressBlogName: Surendhar's Blog!
wordpressEmail: suren@ebi.ac.uk
wordpressFirstName: Surendhar
wordpressLastName: Ravichandran
wordpressSkipInstall: true
wordpressUsername: suren
```

Once the WordPress helm chart is updated, we need to repeat the initial setup.  

Lets test the reliability of the service by restarting the mariadb stateful-set.

```shell
$ kubectl -n wordpress rollout restart statefulset db-mariadb-galera
```

There was a 3-second response time during the mariadb restart, but no downtime was observed in the WordPress pods and 
UptimeRobot.

![Screenshot 2024-03-26 at 20.51.19.png](img%2FScreenshot%202024-03-26%20at%2020.51.19.png)

✅ **Achieved Near-Zero Downtime**

## Conclusion

In this exercise, we have deployed a kubernetes cluster 1.27 using RKE2. We have successfully deployed a WordPress site
on a Kubernetes cluster using Helm Charts. We have also upgraded the cluster to 1.28 version and achieved near-zero 
downtime by deploying a MariaDB Galera Cluster backing the WordPress site. The availability of the WordPress site was 
monitored using UptimeRobot.

### References:
1. [RKE2 Documentation](https://docs.rke2.io/)
2. [Rancher Documentation](https://rancher.com/docs/)
3. [Longhorn Documentation](https://longhorn.io/docs/)
4. [Wordpress Documentation](https://wordpress.org/)
4. [Wordpress Helm Chart](https://github.com/bitnami/charts/tree/main/bitnami/wordpress/)
5. [MariaDB Galera Helm Chart](https://github.com/bitnami/charts/tree/main/bitnami/mariadb-galera/)
