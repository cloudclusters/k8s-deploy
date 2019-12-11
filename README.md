K8s-deploy
===
 [![](https://img.shields.io/badge/platform-linux--64-lightgrey)](https://img.shields.io/badge/platform-linux--64-lightgrey)
 [![](https://img.shields.io/badge/code%20size-764%20Kb-blue)](https://img.shields.io/badge/code%20size-764%20Kb-blue)
 [![](https://img.shields.io/badge/docs-latest-brightgreen.svg)](https://img.shields.io/badge/docs-latest-brightgreen.svg)
 [![](https://img.shields.io/badge/license-MIT-green)](https://img.shields.io/badge/license-MIT-green)
 
<img src="https://www.cloudclusters.io/img/cloudclusters-logo.png" width="100">  

----   

Automatic deployment of `Kubernetes cluster` via ansible script. 
&nbsp;


## Environment

| hostname | Private IP |Intranet card |OS |
| ------ | ------ |------ |------|
| master1 | 192.168.10.148 |ens7 |Ubuntu18.04 |
| master2 |  192.168.10.149 |ens7 |Ubuntu18.04 |
| master3 | 192.168.10.151 |ens7 |Ubuntu18.04 |
| node1 | 192.168.10.152 |ens7 |Ubuntu18.04 |
| node2 | 192.168.10.153 |ens7 |Ubuntu18.04 |
| node3 | 192.168.10.155 |ens7 |Ubuntu18.04 |
| node4 | 192.168.10.156 |ens7 |Ubuntu18.04 |

&nbsp;

## Requirements

- Ansible
- Python 2.7+

&nbsp;

## Installation

1.&nbsp;Log in to master1 and download the deployment file.
```shell
export K8S_INSTALL_DIR="/opt"
cd $K8S_INSTALL_DIR
git clone https://github.com/cloudclusters/k8s-deploy.git
```
2.&nbsp;Install ansible, configure secret-key free login to other nodes on master1.
```shell
apt-get install ansible -y
```

3.&nbsp;Modify ansible config file.
```shell
cd $K8S_INSTALL_DIR/k8s-deploy/deploy
sed -i "s#^inventory.*#inventory = $K8S_INSTALL_DIR/k8s-deploy/deploy/hosts #g" ansible.cfg
sed -i "s#^roles_path.*#roles_path = $K8S_INSTALL_DIR/k8s-deploy/deploy #g" ansible.cfg
```
4.&nbsp; Modify inventory hosts file.
```yaml
# Kubernetes nodes
[master01]
192.168.10.148 host_name=master1

[master02]
192.168.10.149 host_name=master2

[master03]
192.168.10.151 host_name=master3

[nodes]
192.168.10.152 host_name=node1
192.168.10.153 host_name=node2
192.168.10.155 host_name=node3

# Nginx load balancing ("ens7" is the name of the internal network card)
[lb]
192.168.10.148 if="ens7" lb_role=master priority=100 host_name=master1 
192.168.10.149 if="ens7" lb_role=backup priority=90 host_name=master2
192.168.10.151 if="ens7" lb_role=backup priority=80 host_name=master3

[all:vars]
# master1's hostname
m1_hostname=master1
# master1's private network ip 
m1_manager_ip=192.168.10.148

# Same as above
m2_hostname=master2
m2_manager_ip=192.168.10.149

m3_hostname=master3
m3_manager_ip=192.168.10.151

# Software version
docker_version=18.09.3
kubelet_version=1.14.0-00
kubectl_version=1.14.0-00
kubeadm_version=1.14.0-00
kubernetes_version=v1.14.0
etcd_version=v3.3.4
helm_version=v2.14.1
certmanager_version=0.9

# dnsDomain on kubeadm-config.yaml define 
domain="example.com"
# Kubernetes API Server
keepalived_vip="192.168.10.100"

ingress_upload_file_size="512m"

cluster_id="C001"

# Use lb node as etcd node
# No changes to the variables "tmp_nodes" and "etcd_nodes"
tmp_nodes="{% for h in groups['lb'] %}{{ hostvars[h]['host_name'] }}=https://{{ h }}:2380,{% endfor %}"
etcd_nodes="{{ tmp_nodes.rstrip(',') }}"

# Network plugin (calico or flannel)
network_plugin="flannel"
# pod network
pod_network="10.244.0.0/16"

bin_dir="/usr/local/bin"

# The etcd certificates directory on kubernetes master nodes
etcd_ca_dir="/etc/etcd/ssl"


# Kubernetes apps yaml directory
yaml_dir="/root/.kube-init"

# Kubernetes certificates directory
kube_cert_dir="/etc/kubernetes/pki"

# Kubernetes config path
kube_config_dir="/etc/kubernetes"

# Monitor email send define
email_host="mail.example.io:25"
email_user="monitor@example.io"
email_passwd="123456"
email_receivers="user@gamil.com"
```

5.&nbsp;Ansible-playbook performs installation of Kubernetes.
```shell
ansible-playbook deploy.yaml
```
Or you can do it step by step:

<details>
<summary>expand</summary>
<pre><code>
ansible-playbook 01-preinstall.yml
ansible-playbook 02-certificates.yml
ansible-playbook 03-etcd.yml
ansible-playbook 04-lb.yml
ansible-playbook 05-init-master.yml
ansible-playbook 06-other-master.yml
ansible-playbook 07-add-node.yml 
ansible-playbook 08-flannel.yml
ansible-playbook 09-ingress.yml
ansible-playbook 10-helm.yml
ansible-playbook 11-cert-manager.yml 
ansible-playbook 12-monitor.yml
</code></pre>
</details>

&nbsp;


## Add Node 

Add node4 to k8s cluster.

1.&nbsp;Edit k8s inventory hosts file. 
```shell
cd $K8S_INSTALL_DIR/k8s-deploy/deploy/
vim hosts
```

2.&nbsp;Modify the "[nodes]" field, delete the original node, add a new node4.
```yaml
[nodes]
192.168.10.156 host_name=node4
```

3.&nbsp;Execute  07-add-node.yml on master1.
```shell
ansible-playbook 07-add-node.yml
```
