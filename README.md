# Deploy a Production Ready Kubernetes Cluster with [Kubespray](https://github.com/kubernetes-sigs/kubespray/tree/master)

![Kubernetes Logo](https://raw.githubusercontent.com/kubernetes-sigs/kubespray/master/docs/img/kubernetes-logo.png)

If you have questions, check the documentation at [kubespray.io](https://kubespray.io) and join us on the [kubernetes slack](https://kubernetes.slack.com), channel **\#kubespray**.
You can get your invite [here](http://slack.k8s.io/)

#### Comparison

##### Kubespray vs Kops :

Kubespray runs on bare metal and most clouds, using Ansible as its substrate for provisioning and orchestration. [Kops](https://github.com/kubernetes/kops) performs the provisioning and orchestration itself, and as such is less flexible in deployment platforms. For people with familiarity with Ansible, existing Ansible deployments or the desire to run a
Kubernetes cluster across multiple platforms, Kubespray is a good choice. Kops, however, is more tightly integrated with the unique features of the clouds it supports so it could be a better choice if you know that you will only be using one platform for the foreseeable future.

##### Kubespray vs Kubeadm :

[Kubeadm](https://github.com/kubernetes/kubeadm) provides domain Knowledge of Kubernetes clusters' life cycle management, including self-hosted layouts, dynamic discovery services and so on. Had it belonged to the new [operators world](https://coreos.com/blog/introducing-operators.html), it may have been named a "Kubernetes cluster operator". Kubespray however, does generic configuration management tasks from the "OS operators" ansible world, plus some initial K8s clustering (with networking plugins included) and control plane bootstrapping.

Kubespray has started using `kubeadm` internally for cluster creation since v2.3 in order to consume life cycle management domain knowledge from it and offload generic OS configuration things from it, which hopefully benefits both sides.

#
## Step 01: Git clone with specific tag:

#### Requirements:

- **Minimum required version of Kubernetes is v1.25**
- **Ansible v2.14+, Jinja 2.11+ and python-netaddr is installed on the machine that will run Ansible commands**
- The target servers must have **access to the Internet** in order to pull docker images. Otherwise, additional configuration is required (See [Offline Environment](docs/offline-environment.md))
- The target servers are configured to allow **IPv4 forwarding**.
- If using IPv6 for pods and services, the target servers are configured to allow **IPv6 forwarding**.
- The **firewalls are not managed**, you'll need to implement your own rules the way you used to.
    in order to avoid any issue during deployment you should disable your firewall.
- If kubespray is run from non-root user account, correct privilege escalation method
    should be configured in the target servers. Then the `ansible_become` flag
    or command parameters `--become or -b` should be specified.

Hardware:
These limits are safeguarded by Kubespray. Actual requirements for your workload can differ. For a sizing guide go to the [Building Large Clusters](https://kubernetes.io/docs/setup/cluster-large/#size-of-master-and-master-components) guide.

- Master
  - Memory: 1500 MB
- Node
  - Memory: 1024 MB

#### Clone kubespray project with specific tag.
```
git clone -b release-2.27 https://github.com/kubernetes-sigs/kubespray.git
```

#
## Step 02: Installing Ansible and change inventory file:

Kubespray supports multiple ansible versions and ships different `requirements.txt` files for them. Depending on your available python version you may be limited in choosing which ansible version to use.

It is recommended to deploy the ansible version used by kubespray into a python virtual environment.

```bash
python3 -m venv k8s
source k8s/bin/activate
cd $KUBESPRAYDIR
pip install -U -r requirements.txt
```

#### Copy environment variable files for custom installation:
```
# Copy inventory/sample as inventory/my-k8s-inventory
cp -rfp inventory/sample inventory/my-k8s-inventory
```

#### Inventory:
The inventory is composed of 3 groups:

- **kube_node** : list of kubernetes nodes where the pods will run.
- **kube_control_plane** : list of servers where kubernetes control plane components (apiserver, scheduler, controller) will run.
- **etcd**: list of servers to compose the etcd server. You should have at least 3 servers for failover purpose.

Below is a complete inventory example:

```bash
## Configure 'ip' variable to bind kubernetes services on a
## different ip than the default iface
node1 ansible_host=95.54.0.12 ip=10.3.0.1
node2 ansible_host=95.54.0.13 ip=10.3.0.2
node3 ansible_host=95.54.0.14 ip=10.3.0.3
node4 ansible_host=95.54.0.15 ip=10.3.0.4
node5 ansible_host=95.54.0.16 ip=10.3.0.5
node6 ansible_host=95.54.0.17 ip=10.3.0.6

[kube_control_plane]
node1
node2
node3

[etcd]
node1
node2
node3

[kube_node]
node1
node2
node3
node4
node5
node6

[k8s_cluster:children]
kube_node
kube_control_plane
```

#### [Kubespray Ansible tags](https://github.com/kubernetes-sigs/kubespray/blob/release-2.23/docs/ansible.md#ansible-tags): The following tags are defined in playbooks.

##### Example commands for using kubespray ansible tags:

Example command to filter and apply only DNS configuration tasks and skip everything else related to host OS configuration and downloading images of containers:
```
ansible-playbook -i inventory/sample/hosts.ini cluster.yml --tags preinstall,facts --skip-tags=download,bootstrap-os
```

And this play only removes the K8s cluster DNS resolver IP from hosts' /etc/resolv.conf files:
```
ansible-playbook -i inventory/sample/hosts.ini -e dns_mode='none' cluster.yml --tags resolvconf
```

And this prepares all container images locally (at the ansible runner node) without installing or upgrading related stuff or trying to upload container to K8s cluster nodes:
```
ansible-playbook -i inventory/sample/hosts.ini cluster.yml \
    -e download_run_once=true -e download_localhost=true \
    --tags download --skip-tags upload,upgrade
```
Note: use --tags and --skip-tags wise and only if you're 100% sure what you're doing.


#
## Step 03: Change variable file: `group_vars/all/all.yml`

External LB sample config:
```
## External LB example config
apiserver_loadbalancer_domain_name: "your LB Domain"
loadbalancer_apiserver:
  address: 192.168.200.10
  port: 6443
```

Internal loadbalancers for apiservers:
```
## Internal loadbalancers for apiservers
loadbalancer_apiserver_localhost: true
# valid options are "nginx" or "haproxy"
loadbalancer_apiserver_type: nginx
```

Set these proxy values in order to update package manager and docker daemon to use proxies and custom CA for https_proxy if needed
```
http_proxy: "PROXY_ADDRESS:PROXY_PORT"
https_proxy: "PROXY_ADDRESS:PROXY_PORT"
# https_proxy_cert_file: ""
```

Refer to roles/kubespray-defaults/defaults/main.yml before modifying no_proxy
```
# no_proxy: "10.233.0.0/18,10.233.64.0/18"
```

Set true to download and cache container
```
download_container: true
```

Deploy container engine
```
# Set false if you want to deploy container engine manually.
deploy_container_engine: true
```

sysctl_file_path to add sysctl conf to
```
sysctl_file_path: "/etc/sysctl.d/99-sysctl.conf"
```

#
## Step 04: change variable file: `group_vars/all/containerd.yml`
We are using the Containerd for container runtime in Kubernetes.  changed contained variable file.

Registries mirror defined within containerd:
```
# Registries defined within containerd.
containerd_registries_mirrors:
  - prefix: docker.io
    mirrors:
      - host: https://hub.mecan.ir
        capabilities: ["pull", "resolve"]
  - prefix: registry.k8s.io
    mirrors:
      - host: https://k8s.mecan.ir
        capabilities: ["pull", "resolve"]
  - prefix: quay.io
    mirrors:
      - host: https://quay.mecan.ir
        capabilities: ["pull", "resolve"]
```

#
## Step 05: change variable file: `group_vars/all/etcd.yml`

[etcd variables doc](https://github.com/kubernetes-sigs/kubespray/blob/release-2.23/docs/etcd.md)

Directory where etcd data stored:
```
etcd_data_dir: /var/lib/etcd
```

Container runtime
```
## docker for docker, crio for cri-o and containerd for containerd.
## Additionally you can set this to kubeadm if you want to install etcd using kubeadm
## Kubeadm etcd deployment is experimental and only available for new deployments
## If this is not set, container manager will be inherited from the Kubespray defaults
## and not from k8s_cluster/k8s-cluster.yml, which might not be what you want.
## Also this makes possible to use different container manager for etcd nodes.
container_manager: containerd
```

Settings for etcd deployment type
```
# It is possible to deploy etcd with three methods. To change the default deployment method (host), use the etcd_deployment_type variable. Possible values are host, kubeadm, and docker.
etcd_deployment_type: kubeadm
```

To expose metrics on a separate HTTP port, define it in the inventory with:
```
etcd_metrics_port: 2381
```

To fully override metrics exposition urls, define it in the inventory with:
```
etcd_listen_metrics_urls: "http://0.0.0.0:2381"
```

#
## Step 06: chnage variable file: `group_vars/k8s_cluster/addons.yml`


Install Helm on nodes:
```
helm_enabled: true
```

Metrics server deployment and configuration:
```
# Metrics Server deployment
metrics_server_enabled: true
metrics_server_container_port: 10250
metrics_server_kubelet_insecure_tls: true
metrics_server_metric_resolution: 15s
# metrics_server_kubelet_preferred_address_types: "InternalIP,ExternalIP,Hostname"
# metrics_server_host_network: false
metrics_server_replicas: 1
```

Rancher Local Path Provisioner install and config:
```
local_path_provisioner_enabled: true
local_path_provisioner_namespace: "local-path-storage"
local_path_provisioner_storage_class: "local-path"
local_path_provisioner_reclaim_policy: Delete
local_path_provisioner_claim_root: /var/lib/local-path-provisioner/
local_path_provisioner_debug: false
local_path_provisioner_image_repo: "registry.mecan.ir/rancher/local-path-provisioner"
local_path_provisioner_image_tag: "v0.0.24"
local_path_provisioner_helper_image_repo: "busybox"
local_path_provisioner_helper_image_tag: "latest"
```

#
## Step 07: chnage variable file: `group_vars/k8s_cluster/k8s-cluster.yml`


Change this to use another Kubernetes version, e.g. a current beta release:
```
kube_version: v1.29.7
```

Choose network plugin (cilium, calico, kube-ovn, weave or flannel. Use cni for generic cni plugin)
Can also be set to 'cloud', which lets the cloud provider setup appropriate routing
```
kube_network_plugin: calico
```

Kubernetes internal network for services and pods.
```
# Kubernetes internal network for services, unused block of space.
kube_service_addresses: 10.233.0.0/18

# internal network. When used, it will assign IP
# addresses from this range to individual pods.
# This network must be unused in your network infrastructure!
kube_pods_subnet: 10.233.64.0/18
```

Kube-proxy proxyMode configuration:
```
# Can be ipvs, iptables
kube_proxy_mode: iptables
```

Graceful Node Shutdown:
```
# Graceful Node Shutdown (Kubernetes >= 1.21.0), see https://kubernetes.io/blog/2021/04/21/graceful-node-shutdown-beta/
# kubelet_shutdown_grace_period had to be greater than kubelet_shutdown_grace_period_critical_pods to allow
# non-critical podsa to also terminate gracefully
kubelet_shutdown_grace_period: 60s
kubelet_shutdown_grace_period_critical_pods: 20s
```

Container runtime:
```
## docker for docker, crio for cri-o and containerd for containerd.
## Default: containerd
container_manager: containerd
```

K8s image pull policy (imagePullPolicy):
```
k8s_image_pull_policy: IfNotPresent
```

audit log for kubernetes:
```
kubernetes_audit: true
```

Make a copy of kubeconfig on the host that runs Ansible in {{ inventory_dir }}/artifacts
```
kubeconfig_localhost: true
```

Optionally reserve this space for kube daemons:
```
kube_reserved: true
## Uncomment to override default values
## The following two items need to be set when kube_reserved is true
# kube_reserved_cgroups_for_service_slice: kube.slice
# kube_reserved_cgroups: "/{{ kube_reserved_cgroups_for_service_slice }}"
kube_memory_reserved: 256Mi
kube_cpu_reserved: 100m
# kube_ephemeral_storage_reserved: 2Gi
kube_pid_reserved: "1000"
# Reservation for master hosts
kube_master_memory_reserved: 512Mi
kube_master_cpu_reserved: 200m
kube_master_ephemeral_storage_reserved: 2Gi
kube_master_pid_reserved: "1000"
```

Optionally reserve resources for OS system daemons:
```
system_reserved: true
## Uncomment to override default values
## The following two items need to be set when system_reserved is true
# system_reserved_cgroups_for_service_slice: system.slice
# system_reserved_cgroups: "/{{ system_reserved_cgroups_for_service_slice }}"
system_memory_reserved: 512Mi
system_cpu_reserved: 500m
system_ephemeral_storage_reserved: 2Gi
## Reservation for master hosts
system_master_memory_reserved: 256Mi
system_master_cpu_reserved: 250m
system_master_ephemeral_storage_reserved: 2Gi
```

Supplementary addresses that can be added in kubernetes ssl keys:
```
## That can be useful for example to setup a keepalived virtual IP
supplementary_addresses_in_ssl_keys:
  - 192.168.200.10
  - 192.168.200.11
  - 192.168.200.12
  - 192.168.200.13
  - vip.kube.mecan.ir
  - master1
  - master2
  - master3
  - master1.kube.mecan.ir
  - master2.kube.mecan.ir
  - master3.kube.mecan.ir
```

Support tls min version, Possible values: VersionTLS10, VersionTLS11, VersionTLS12, VersionTLS13:
```
tls_min_version: "VersionTLS12"
```

Automatically renew K8S control plane certificates on first Monday of each month:
```
auto_renew_certificates: true
# First Monday of each month
auto_renew_certificates_systemd_calendar: "Mon *-*-1,2,3,4,5,6,7 03:{{ groups['kube_control_plane'].index(inventory_hostname) }}0:00"
```

system upgrade configuration
```
system_upgrade: true
system_upgrade_reboot: never
```

#
## Step 08: chnage variable file: `group_vars/k8s_cluster/k8s-net-calico.yml`

If you want to use non default IP_AUTODETECTION_METHOD, IP6_AUTODETECTION_METHOD for calico node set this option to one of:
```
# * can-reach=DESTINATION
# * interface=INTERFACE-REGEX
# see https://docs.projectcalico.org/reference/node/configuration
calico_ip_auto_method: "interface=eth.*"
calico_ip6_auto_method: "interface=eth.*"
```

Choose the iptables insert mode for Calico: "Insert" or "Append".
```
calico_felix_chaininsertmode: Insert
```

Enable calico traffic encryption with wireguard
```
calico_wireguard_enabled: false
```

Under certain situations liveness and readiness probes may need tunning
```
calico_node_livenessprobe_timeout: 10
calico_node_readinessprobe_timeout: 30
```

#
## Step 09: run ansible playbook

Before running Ansible, ensure that access is allowed in iptables for Kubernetes cluster connectivity, and persist all rules in the iptables rule file.

```
iptables -A INPUT -s 192.168.200.10/32 -j ACCEPT -m comment --comment "The Trusted lb server"
iptables -A INPUT -s 192.168.200.11/32 -j ACCEPT -m comment --comment "The Trusted master1 server"
iptables -A INPUT -s 192.168.200.12/32 -j ACCEPT -m comment --comment "The Trusted master2 server"
iptables -A INPUT -s 192.168.200.13/32 -j ACCEPT -m comment --comment "The Trusted master3 server"
iptables -A INPUT -s 192.168.200.14/32 -j ACCEPT -m comment --comment "The Trusted worker1 server"
iptables -A INPUT -s 192.168.200.15/32 -j ACCEPT -m comment --comment "The Trusted worker2 server"
iptables -A INPUT -s 192.168.200.16/32 -j ACCEPT -m comment --comment "The Trusted worker3 server"
iptables -A INPUT -s 10.233.0.0/18 -j ACCEPT -m comment --comment "Kubernetes internal network for services"
iptables -A INPUT -s 10.233.64.0/18 -j ACCEPT -m comment --comment "Kubernetes internal network for pod"
```




The second step deploy Kubespray with Ansible Playbook - run the playbook as root:
```
ansible-playbook -i inventory/my-k8s-inventory/inventory.ini --become --become-user=root cluster.yml
```

Now we will copy over the kubeconfig file:
```
cat /etc/kubernetes/admin.conf > ~/.kube/config
```


#
## Scale your cluster:
You can add worker nodes from your cluster by running the scale playbook. For more information, see "Adding nodes". You can remove worker nodes from your cluster by running the remove-node playbook. For more information, see "Remove nodes".


[Add node full documents](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/nodes.md)

#### Adding a worker node
This should be the easiest.

1) Add new node to the `ansible/inventory/invrntory.ini`

 2) Run Preparing ansible with this command

```
cd ansible
ansible-playbook -i inventory/hosts.yml playbook/preparing.yaml --limit=worker3
```

3) Add new node to the `kubespray/inventory/my-k8s-inventory/inventory.ini`

4) Run `scale.yml`
Before using `--limit` run playbook `facts.yml` without the limit to refresh facts cache for all nodes.
```bash
cd kubespray
ansible-playbook -i inventory/MeCan/inventory.ini facts.yml
```

You can use `--limit=NODE_NAME` to limit Kubespray to avoid disturbing other nodes in the cluster.
```bash
ansible-playbook -i inventory/MeCan/inventory.ini scale.yml --limit=worker3
```

#### Remove nodes:
You may want to remove control plane, worker, or etcd nodes from your existing cluster. This can be done by re-running the remove-node.yml playbook. First, all specified nodes will be drained, then stop some kubernetes services and delete some certificates, and finally execute the kubectl command to delete these nodes. This can be combined with the add node function. This is generally helpful when doing something like autoscaling your clusters. Of course, if a node is not working, you can remove the node and install it again.

Use `--extra-vars "node=<nodename>,<nodename2>"` to select the node(s) you want to delete.

```
ansible-playbook -i inventory/mycluster/hosts.yml remove-node.yml -b -v \
--private-key=~/.ssh/private_key \
--extra-vars "node=nodename,nodename2"
```
If a node is completely unreachable by ssh, add `--extra-vars reset_nodes=false` to skip the node reset step. If one node is unavailable, but others you wish to remove are able to connect via SSH, you could set `reset_nodes=false` as a host var in inventory.

#
## Upgrading Kubernetes in Kubespray. [link](https://github.com/kubernetes-sigs/kubespray/blob/release-2.23/docs/upgrades.md#upgrading-kubernetes-in-kubespray)

Kubespray handles upgrades the same way it handles initial deployment. That is to say that each component is laid down in a fixed order.

You can also individually control versions of components by explicitly defining their versions. Here are all version vars for each component:

- docker_version
- docker_containerd_version (relevant when container_manager == docker)
- containerd_version (relevant when container_manager == containerd)
- kube_version
- etcd_version
- calico_version
- calico_cni_version
- weave_version
- flannel_version
- kubedns_version

#### Graceful upgrade:
Kubespray also supports cordon, drain and uncordoning of nodes when performing a cluster upgrade. There is a separate playbook used for this purpose. It is important to note that upgrade-cluster.yml can only be used for upgrading an existing cluster. That means there must be at least 1 kube_control_plane already deployed.
```
ansible-playbook upgrade-cluster.yml -b -i inventory/sample/hosts.ini -e kube_version=v1.19.7
```
After a successful upgrade, the Server Version should be updated:
```
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.7", GitCommit:"1dd5338295409edcfff11505e7bb246f0d325d15", GitTreeState:"clean", BuildDate:"2021-01-13T13:23:52Z", GoVersion:"go1.15.5", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.7", GitCommit:"1dd5338295409edcfff11505e7bb246f0d325d15", GitTreeState:"clean", BuildDate:"2021-01-13T13:15:20Z", GoVersion:"go1.15.5", Compiler:"gc", Platform:"linux/amd64"}
```

#### Node-based upgrade:
If you don't want to upgrade all nodes in one run, you can use `--limit` patterns.

Before using `--limit` run playbook facts.yml without the limit to refresh facts cache for all nodes:

```
ansible-playbook facts.yml -b -i inventory/sample/hosts.ini
```

After this upgrade control plane and etcd groups #5147:
```
ansible-playbook upgrade-cluster.yml -b -i inventory/sample/hosts.ini -e kube_version=v1.20.7 --limit "kube_control_plane:etcd"
```
Now you can upgrade other nodes in any order and quantity:
```
ansible-playbook upgrade-cluster.yml -b -i inventory/sample/hosts.ini -e kube_version=v1.20.7 --limit "node4:node6:node7:node12"
ansible-playbook upgrade-cluster.yml -b -i inventory/sample/hosts.ini -e kube_version=v1.20.7 --limit "node5*"
```

#### Upgrade order:
As mentioned above, components are upgraded in the order in which they were installed in the Ansible playbook. The order of component installation is as follows:

- Docker
- Containerd
- etcd
- kubelet and kube-proxy
- network_plugin (such as Calico or Weave)
- kube-apiserver, kube-scheduler, and kube-controller-manager
- Add-ons (such as KubeDNS)

###### Component-based upgrades
A deployer may want to upgrade specific components in order to minimize risk or save time. This strategy is not covered by CI as of this writing, so it is not guaranteed to work.

These commands are useful only for upgrading fully-deployed, healthy, existing hosts. This will definitely not work for undeployed or partially deployed hosts.

Upgrade docker:
```
ansible-playbook -b -i inventory/sample/hosts.ini cluster.yml --tags=docker
```

Upgrade etcd:
```
ansible-playbook -b -i inventory/sample/hosts.ini cluster.yml --tags=etcd
```

Upgrade etcd without rotating etcd certs:
```
ansible-playbook -b -i inventory/sample/hosts.ini cluster.yml --tags=etcd --limit=etcd --skip-tags=etcd-secrets
```

Upgrade kubelet:
```
ansible-playbook -b -i inventory/sample/hosts.ini cluster.yml --tags=node --skip-tags=k8s-gen-certs,k8s-gen-tokens
```

Upgrade Kubernetes master components:
```
ansible-playbook -b -i inventory/sample/hosts.ini cluster.yml --tags=master
```

Upgrade network plugins:
```
ansible-playbook -b -i inventory/sample/hosts.ini cluster.yml --tags=network
```

Upgrade all add-ons:
```
ansible-playbook -b -i inventory/sample/hosts.ini cluster.yml --tags=apps
```

Upgrade just helm (assuming helm_enabled is true):
```
ansible-playbook -b -i inventory/sample/hosts.ini cluster.yml --tags=helm
```

#### System upgrade:
If you want to upgrade the APT or YUM packages while the nodes are cordoned, you can use:
```
ansible-playbook upgrade-cluster.yml -b -i inventory/sample/hosts.ini -e system_upgrade=true
```
Nodes will be rebooted when there are package upgrades (system_upgrade_reboot: on-upgrade). This can be changed to always or never.

**Note:** Downloads will happen twice unless system_upgrade_reboot is never.
