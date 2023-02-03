## Supported hairstyles

- CentOS/RHEL 7，8
- Ubuntu 18.04，20.04，22.04
- Alma Linux 8
- Rocky Linux 8



## Support components

- Core
  - [kubernetes](https://github.com/kubernetes/kubernetes)
  - [etcd](https://github.com/etcd-io/etcd)
  - [containerd](https://github.com/containerd/containerd)
- Network Plugin
  - [cni-plugins](https://github.com/containernetworking/plugins)
  - [calico](https://github.com/projectcalico/calico)
  - [cilium](https://github.com/cilium/cilium)
  - [flanneld](https://github.com/flannel-io/flannel)
- Application
  - [coredns](https://github.com/coredns/coredns)
  - [metrics-server](https://github.com/kubernetes-sigs/metrics-server)
  - [kubernetes-dashboard](https://github.com/kubernetes/dashboard)
  - [helm](https://github.com/helm/helm)



## Basic configuration

### Install Ansible

```
pip3 install ansible
pip3 install netaddr -i  https://mirrors.ustc.edu.cn/pypi/web/simple
```

- If you use Python3, please add `interpreter_python = /usr/bin/python3` under the defaults configuration of ansible.cfg.
- Try to keep the Python version of the control node and the controlled node consistent, otherwise problems may occur in execution.


### Modify inventory

Please modify the corresponding resources according to the inventory template format

- When haproxy and kube-apiserver are deployed on the same server, please ensure that the ports do not conflict.

### Mount the data disk

If you have already formatted and mounted the directory yourself, you can skip this step.

```
ansible-playbook fdisk.yml -i inventory -e "disk=sdb dir=/data"
```

If it is an NVME disk, please use the following method:

```
ansible-playbook fdisk.yml -i inventory -e "disk=sdb dir=/data type=nvme"
```
⚠️：

- This script will format the hard disk specified by {{disk}} and mount it to the {{dir}} directory.
- Will bind `/var/lib/etcd`, `/var/lib/containerd`, `/var/lib/kubelet`, `/var/log/pods` data directories to this data disk`{{dir }}/containers/etcd`, `{{dir}}/containers/containerd`, `{{dir}}/containers/kubelet`, `{{dir}}/containers/pods` directories to reach multiple data The directory shares a data disk, without modifying the kubernetes-related data directory.


If you need to mount different data disks in different directories, you can use the following commands to mount them separately

```
ansible-playbook fdisk.yml -i inventory -l master -e "disk=sdb dir=/var/lib/etcd" --skip-tags=bind_dir
```

If the data disk has been formatted and mounted, you can use the following command to bind the data directory to the data disk

```
ansible-playbook fdisk.yml -i inventory -l master -e "disk=sdb dir=/data" -t bind_dir
```



### Configure group_vars

Edit the group_vars/all.yml file and fill in your own configuration.

Please note:

- Please try to install etcd on an independent server, it is not recommended to install it together with the master. Use SSD disks as much as possible for data disks.
- The Pod and Service IP network segments are recommended to use reserved private IP segments. It is recommended (the Pod IP should not overlap with the Service IP, nor with the host IP segment, and also avoid conflicts with the network segment of the docker0 network card.):
- Pod segment
    - Class A address: 10.0.0.0/8
    - Class B address: 172.16-31.0.0/12-16
    - Class C address: 192.168.0.0/16
  - Service network segment
    - Class A address: 10.0.0.0/16-24
    - Class B address: 172.16-31.0.0/16-24
    - Class C address: 192.168.0.0/16-24



## Deploy the cluster

Format and mount the data disk

```
ansible-playbook fdisk.yml -i inventory -e "disk=sdb dir=/data"
```

deploy cluster

```
ansible-playbook cluster.yml -i inventory
```

If it is a public cloud environment, you can use the load balancing of the public cloud (the load balancing needs to be configured in advance), and there is no need to install haproxy and keepalived.

```
ansible-playbook cluster.yml -i inventory --skip-tags=haproxy,keepalived
```
- By default, the node will be initialized, and the cluster node will take the last two segments of the host name and the IP as the cluster node name.

If you want the master node to also schedule, you can add the following methods

```
ansible-playbook cluster.yml -i inventory --skip-tags=create_master_taint
```



## Expansion node

### Expand the master node

When expanding capacity, it is recommended to comment out the old server information in the master group of the inventory file, and only keep the information of the expanded node.

Format and mount the data disk

```
ansible-playbook fdisk.yml -i inventory -l ${SCALE_MASTER_IP} -e "disk=sdb dir=/data"
```
Execute Generate Node Certificate

```
ansible-playbook cluster.yml -i inventory -t cert
```

Perform node initialization

```
ansible-playbook cluster.yml -i inventory -l ${SCALE_MASTER_IP} -t verify,init
```
Execute node expansion

```
ansible-playbook cluster.yml -i inventory -l ${SCALE_MASTER_IP} -t master,containerd,worker --skip-tags=bootstrap,create_worker_label
```



### Expansion of worker nodes

When expanding capacity, it is recommended to comment out the old server information in the worker group in the inventory file, and only keep the expanded node information.

Format and mount the data disk

```
ansible-playbook fdisk.yml -i inventory -l ${SCALE_MASTER_IP} -e "disk=sdb dir=/data"
```

Execute Generate Node Certificate

```
ansible-playbook cluster.yml -i inventory -t cert
```

Perform node initialization

```
ansible-playbook cluster.yml -i inventory -l ${SCALE_WORKER_IP} -t verify,init
```
Execute node expansion

```
ansible-playbook cluster.yml -i inventory -l ${SCALE_WORKER_IP} -t containerd,worker --skip-tags=bootstrap,create_master_label
```



## Replace the cluster certificate

First backup and delete the certificate directory {{cert.dir}}, recreate {{cert.dir}}, and copy the token, sa.pub, sa.key files to the newly created {{cert.dir}} (this The three files must be kept and cannot be changed), and then perform the following steps to regenerate the certificate and distribute the certificate.

```
ansible-playbook cluster.yml -i inventory -t cert,dis_certs
```

Then restart each node in turn.

restart etcd

```
ansible -i inventory etcd -m systemd -a "name=etcd state=restarted"
```

Verify etcd

```
etcdctl endpoint health \
        --cacert=/etc/etcd/pki/etcd-ca.pem \
        --cert=/etc/etcd/pki/etcd-healthcheck-client.pem \
        --key=/etc/etcd/pki/etcd-healthcheck-client.key \
        --endpoints=https://172.16.90.101:2379,https://172.16.90.102:2379,https://172.16.90.103:2379
```

Remove old kubelet certificates one by one

```
ansible -i inventory -l master,worker -m shell -a "rm -rf /etc/kubernetes/pki/kubelet*"
```

- The `-l` parameter is replaced with the specific node IP.

Restart nodes one by one

```
ansible-playbook cluster.yml -i inventory -l ${IP} -t restart_apiserver,restart_controller,restart_scheduler,restart_kubelet,restart_proxy,healthcheck
```
- Services such as calico and metrics-server also use cluster certificates, please remember to update related certificates together.
- The `-l` parameter is replaced with the specific node IP.

Restart the network plugin

```
kubectl get pod -n kube-system | grep -v NAME | grep cilium | awk '{print $1}' | xargs kubectl -n kube-system delete pod
```
- Updating the certificate may cause the network plug-in to be abnormal, it is recommended to restart.
- The example is the command to restart the cilium plug-in, please replace it according to different network plug-ins.

## Upgrade kubernetes version
Please edit group_vars/all.yml first, and modify kubernetes.version to the new version.

Install kubernetes components

```
ansible-playbook cluster.yml -i inventory -t install_kubectl,install_master,install_worker
```
update configuration file

```
ansible-playbook cluster.yml -i inventory -t dis_master_config,dis_worker_config
```
Then restart each kubernetes component in turn.

```
ansible-playbook cluster.yml -i inventory -l ${IP} -t restart_apiserver,restart_controller,restart_scheduler,restart_kubelet,restart_proxy,healthcheck
```
- The `-l` parameter is replaced with the specific node IP.
