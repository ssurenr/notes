# Kubernetes Cluster Deployment and Management with RKE2

## Cluster Setup and Initial Configuration

### Connection Details

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
### Node Details

| Node      | Role    | IP             | CPU (number) | Memory (GB) | Storage (GB) |
|-----------|---------|----------------|--------------|-------------|--------------|
| bastion-1 | bastion | 45.88.80.253   | 4            | 8           | 80           |
| master-1  | master  | 192.168.10.129 | 4            | 8           | 80           |
| worker-1  | worker  | 192.168.10.97  | 4            | 8           | 80           |


## Step 1: Install Rancher

### Install Docker

sudo snap install kubectl --classic
kubectl 1.29.3 from Canonical✓ installed


```shell
$ sudo apt install docker.io
```
```shell
docker run --privileged -d --name rancher --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher:v2.8.3-rc6
4fd5bff9f44243487146f4bb7e973d8abb27a92d8a18b8fcef1f87a2284cbb0a
```

```shell
docker logs rancher 2>&1 | grep "Bootstrap Password:"
2024/03/24 23:52:17 [INFO] Bootstrap Password: r5hf9bxhh66p4fwqdj974mnbqwnlqvlk8p6tfhgx5v2hbfl5tt5587
```

### Bootstram Master

```shell
curl --insecure -fL https://45.88.80.253/system-agent-install.sh | sudo  sh -s - --server https://45.88.80.253 --label 'cattle.io/os=linux' --token x6s8w9st22jf6j9lrcbgrn2q528j5f962s6flkzdk5j9gl9gqnz2b8 --ca-checksum c8019bafeade0741fd36e31373d6cb4d43cc62146bff4df661bfb140bb8bf998 --etcd --controlplane --worker
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

### Bootstrap Worker

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

### Monitoring

```shell
helm upgrade --install=true --namespace=cattle-monitoring-system --timeout=10m0s --values=/home/shell/helm/values-rancher-monitoring-crd-103.0.4-up45.31.1.yaml --version=103.0.4+up45.31.1 --wait=true rancher-monitoring-crd /home/shell/helm/rancher-monitoring-crd-103.0.4-up45.31.1.tgz
```

### Persistent Volume

```shell
helm upgrade --install=true --namespace=longhorn-system --timeout=10m0s --values=/home/shell/helm/values-longhorn-crd-103.2.2-up1.5.4.yaml --version=103.2.2+up1.5.4 --wait=true longhorn-crd /home/shell/helm/longhorn-crd-103.2.2-up1.5.4.tgz
2024-03-25T01:00:36.701468690Z Release "longhorn-crd" does not exist. Installing it now.
```

## Provisioning Wordpress

Inspect wordpress helm chart

```shell
helm template my-release oci://registry-1.docker.io/bitnamicharts/wordpress
```
```shell
helm install --namespace=wordpress --timeout=10m0s --values=/home/shell/helm/values-wordpress-20.1.2.yaml --version=20.1.2 --wait=true wordpress /home/shell/helm/wordpress-20.1.2.tgz
```

### Scale Wordpress

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
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
Deployment is not ready: wordpress/wordpress. 1 out of 2 expected pods are ready
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

### Errors

#### NFS Installation

```shell
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
Checked longhorn documentation. For RWX volumes, the NFS client must be installed on the nodes. Installed NFS client on 
the master and worker nodes.

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
$ kubectl -n wordpress get pods -l app.kubernetes.io/name=wordpress
NAME                         READY   STATUS    RESTARTS   AGE
wordpress-65cb4497bf-49chv   1/1     Running   0          25m
wordpress-65cb4497bf-l7cp6   1/1     Running   0          13m
```