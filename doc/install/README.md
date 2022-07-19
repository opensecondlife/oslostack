# 快速开始

本指南提供了使用 ceph-ansible 与 kolla-ansible 在裸金属服务器或虚拟机上部署 Ceph 与 OpenStack 的逐步说明。

## 交换机配置

略

## 安装操作系统

请参考 [How to Install CentOS 8 Stream](https://linuxhint.com/install_centos8_stream/)。

- 使用默认的 LVM 分区方案。

- 除必要分区外，将剩余可用容量空间全部添加到 **/** 分区中。

- 无需添加 swap 分区。

- 在选择软件环境窗口选择最小化安装（Minimal Install）。

## 准备工作

### 在所有节点上进行

#### 准备 Linux 发行版/软件的安装源

> :warning: 操作前请做好备份。

对于 CentOS 8 Stream，使用以下命令替换默认的配置

```shell
sudo sed -e 's|^mirrorlist=|#mirrorlist=|g' \
         -e 's|^#baseurl=http://mirror.centos.org/$contentdir|baseurl=https://mirrors.ustc.edu.cn/centos|g' \
         -i.bak \
         /etc/yum.repos.d/CentOS-Stream-AppStream.repo \
         /etc/yum.repos.d/CentOS-Stream-BaseOS.repo \
         /etc/yum.repos.d/CentOS-Stream-Extras.repo \
         /etc/yum.repos.d/CentOS-Stream-PowerTools.repo
```

查看替换后的文件

```shell
[root@oslostack01 ~]# cat /etc/yum.repos.d/CentOS-Stream-BaseOS.repo
# CentOS-Stream-BaseOS.repo
#
# The mirrorlist system uses the connecting IP address of the client and the
# update status of each mirror to pick current mirrors that are geographically
# close to the client.  You should use this for CentOS updates unless you are
# manually picking other mirrors.
#
# If the mirrorlist does not work for you, you can try the commented out
# baseurl line instead.

[baseos]
name=CentOS Stream $releasever - BaseOS
#mirrorlist=http://mirrorlist.centos.org/?release=$stream&arch=$basearch&repo=BaseOS&infra=$infra
baseurl=https://mirrors.ustc.edu.cn/centos/$stream/BaseOS/$basearch/os/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial
[root@oslostack01 ~]# cat /etc/yum.repos.d/CentOS-Stream-Extras.repo
# CentOS-Stream-Extras.repo
#
# The mirrorlist system uses the connecting IP address of the client and the
# update status of each mirror to pick current mirrors that are geographically
# close to the client.  You should use this for CentOS updates unless you are
# manually picking other mirrors.
#
# If the mirrorlist does not work for you, you can try the commented out
# baseurl line instead.

[extras]
name=CentOS Stream $releasever - Extras
#mirrorlist=http://mirrorlist.centos.org/?release=$stream&arch=$basearch&repo=extras&infra=$infra
baseurl=https://mirrors.ustc.edu.cn/centos/$stream/extras/$basearch/os/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial
[root@oslostack01 ~]# cat /etc/yum.repos.d/CentOS-Stream-AppStream.repo
# CentOS-Stream-AppStream.repo
#
# The mirrorlist system uses the connecting IP address of the client and the
# update status of each mirror to pick current mirrors that are geographically
# close to the client.  You should use this for CentOS updates unless you are
# manually picking other mirrors.
#
# If the mirrorlist does not work for you, you can try the commented out
# baseurl line instead.

[appstream]
name=CentOS Stream $releasever - AppStream
#mirrorlist=http://mirrorlist.centos.org/?release=$stream&arch=$basearch&repo=AppStream&infra=$infra
baseurl=https://mirrors.ustc.edu.cn/centos/$stream/AppStream/$basearch/os/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial
[root@oslostack01 ~]# cat /etc/yum.repos.d/CentOS-Stream-PowerTools.repo
# CentOS-Stream-PowerTools.repo
#
# The mirrorlist system uses the connecting IP address of the client and the
# update status of each mirror to pick current mirrors that are geographically
# close to the client.  You should use this for CentOS updates unless you are
# manually picking other mirrors.
#
# If the mirrorlist does not work for you, you can try the commented out
# baseurl line instead.

[powertools]
name=CentOS Stream $releasever - PowerTools
#mirrorlist=http://mirrorlist.centos.org/?release=$stream&arch=$basearch&repo=PowerTools&infra=$infra
baseurl=https://mirrors.ustc.edu.cn/centos/$stream/PowerTools/$basearch/os/
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial
```

#### 网络配置

TODO

## 在部署节点上进行

### 安装发行版的基础依赖包

```shell
dnf install -y git vim python3 tmux python3-devel libffi-devel gcc openssl-devel python3-libselinux python3-netaddr
```

> :warning: 后续操作步骤可选择在 tmux 终端中进行，避免会话丢失造成部署过程中断。

### 创建安装目录

```shell
mkdir -p /opt/oslostack && cd /opt/oslostack
```

### 部署介质列表

可以提前把部分安装介质准备好，以加快部署速度。

- ceph-ansible：[GitHub - ceph/ceph-ansible: Ansible playbooks to deploy Ceph, the distributed filesystem.](https://github.com/ceph/ceph-ansible)

- kolla-ansible：[https://opendev.org/openstack/kolla-ansible.git](https://opendev.org/openstack/kolla-ansible.git)

- kolla容器镜像：[Docker Hub Kolla](https://hub.docker.com/u/kolla)

### 获取部署脚本

```shell
git clone https://github.com/ceph/ceph-ansible.git
git clone https://opendev.org/openstack/kolla-ansible.git
```

### 准备 python 虚拟环境

```shell
python3 -m venv /opt/oslostack/venv
source /opt/oslostack/venv/bin/activate
```

### 安装 python 依赖

```shell
pip install -U pip 
pip install 'ansible<2.10'
```

### 安装 kolla-ansible

```shell
cd ./kolla-ansible
git checkout stable/wallaby
pip install .
cd ..
mkdir -p /etc/kolla
cp ./venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
```

### 填写 /etc/hosts

```shell
vim /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.10.34 oslostack01
192.168.10.243 oslostack02
192.168.10.107 oslostack03
```

### 修改 ansible 配置文件

```shell
vim /etc/ansible/ansible.cfg
[defaults]
host_key_checking=False
pipelining=True
forks=100
```

### 填写 ansible inventory

control 组指控制节点，主要运行 OpenStack 控制层面的服务

compute 组指计算节点，主要运行 hypervisor 以及 虚拟机工作负载

mons 组指 ceph mon 节点，主要运行 ceph mon 管理服务

osds 组指 ceph osd 节点，主要运行 ceph osd 服务

```shell
cp ./kolla-ansible/ansible/inventory/multinode ./inventory
vim ./inventory
[all:vars]
# Host user name and password must be required if no ssh trust
ansible_connection=ssh
#
# Using password
ansible_user=root
ansible_ssh_pass=******

# These initial groups are the only groups required to be modified. The
# additional groups are for more control of the environment.
[control]
# These hostname must be resolvable from your deployment host
oslostack[01:03]

# The above can also be specified as follows:
#control[01:03]     ansible_user=kolla

# The network nodes are where your l3-agent and loadbalancers will run
# This can be the same as a host in the control group
[network:children]
control

[compute]
oslostack[01:03]

[monitoring]
# monitoring01

# When compute nodes and control nodes use different interfaces,
# you need to comment out "api_interface" and other interfaces from the globals.yml
# and specify like below:
#compute01 neutron_external_interface=eth0 api_interface=em1 storage_interface=em1 tunnel_interface=em1

[storage:children]
control

[mons:children]
control

[osds:children]
compute
```

### 测试节点连通性

```shell
ansible -i ./inventory all -m ping
```

## 使用 ceph-ansible 部署 ceph 分布式存储

本章节提供了 ceph 分布式存储的部署步骤以及相关指令的说明。

### 安装 ceph-ansible

#### 克隆仓库

```shell
git clone https://github.com/ceph/ceph-ansible.git
```

#### 版本说明

- stable-3.2 支持 Ceph luminous 和 mimic 版本。这个分支需要 Ansible 2.6 版本。

- stable-4.0 支持 Ceph nautilus 版本。这个分支需要 Ansible 2.9 版本。

- stable-5.0 支持 Ceph octopus 版本。这个分支需要 Ansible 2.9 版本。

- stable-6.0 支持 Ceph pacific 版本。这个分支需要 Ansible 2.9 版本。

- stable-master 支持 Ceph master 分支 版本。这个分支需要 Ansible 2.10 版本。

#### 切换到您希望使用的某个版本分支

```shell
cd ceph-ansible
git checkout stable-6.0
```

### 填写 ceph-ansible 配置

```shell
cd /opt/oslostack
vim oslostack.yml
```

```yml
# ceph
cephx: true
containerized_deployment: true
ceph_origin: distro

osd_scenario: lvm
osd_objectstore: bluestore
osd_auto_discovery: true
# 显示设置 db 空间大小，单位为 bytes，默认 -1 为平分空间
block_db_size: -1

public_network: "192.168.3.0/24"
cluster_network: "192.168.4.0/24"
monitor_interface: eth1

ntp_service_enabled: false

ceph_mgr_modules:
  - status
  - restful
  - balancer
  - diskprediction_local
  - iostat
  - pg_autoscaler
  - prometheus

crush_rule_config: true
crush_rule_hdd:
  name: hdd
  root: default
  type: host
  default: false
  class: hdd
crush_rule_ssd:
  name: ssd
  root: default
  type: host
  default: false
  class: ssd
crush_rules:
  - "{{ crush_rule_hdd }}"
  - "{{ crush_rule_ssd }}"

openstack_config: true
openstack_glance_pool:
  name: "images"
  application: "rbd"
  pg_autoscale_mode: "on"
  target_size_ratio: 0.2
openstack_nova_pool:
  name: "vms"
  application: "rbd"
  pg_autoscale_mode: "on"
  target_size_ratio: 0.1
openstack_cinder_ssd_pool:
  name: "ssd-volumes"
  rule_name: "{{ crush_rule_ssd.name }}"
  application: "rbd"
  pg_autoscale_mode: "on"
  target_size_ratio: 1.0
openstack_cinder_hdd_pool:
  name: "hdd-volumes"
  rule_name: "{{ crush_rule_hdd.name }}"
  application: "rbd"
  pg_autoscale_mode: "on"
  target_size_ratio: 1.0
openstack_pools:
  - "{{ openstack_glance_pool }}"
  - "{{ openstack_cinder_ssd_pool }}"
  - "{{ openstack_cinder_hdd_pool }}"
  - "{{ openstack_nova_pool }}"
openstack_keys:
  - { name: client.glance, caps: { mon: "profile rbd", osd: "profile rbd"}, mode: "0600" }
  - { name: client.cinder, caps: { mon: "profile rbd", osd: "profile rbd"}, mode: "0600" }
  - { name: client.nova, caps: { mon: "profile rbd", osd: "profile rbd"}, mode: "0600" }

```

### 执行 ceph-ansible 部署命令

```shell
cd /opt/oslostack
ansible-playbook -i ./inventory ./ceph-ansible/site-container.yml.sample -e @/opt/oslostack/oslostack.yml
```

## 使用 kolla-ansible 部署 OpenStack

### 生成 kolla 密码

```shell
kolla-genpwd
```

### 填写 kolla-ansible 配置

```shell
cd /opt/oslostack
vim oslostack.yml
```

```yml
kolla_base_distro: "centos"
kolla_install_type: "source"

# 外部网络
network_interface: "eth0"
neutron_external_interface: "eth1"
kolla_internal_vip_address: "10.1.0.250"
enable_cinder: "yes"
enable_cinder_backup: "no"
enable_fluentd: "no"
```

### 填写 OpenStack 与 Ceph 对接配置

```shell
mkdir -p /etc/kolla/config/glance
mkdir -p /etc/kolla/config/cinder
mkdir -p /etc/kolla/config/nova
cp ceph.client.glance.keyring /etc/kolla/config/glance/
cp ceph.client.cinder.keyring /etc/kolla/config/cinder/
cp ceph.client.nova.keyring /etc/kolla/config/nova/

vim /opt/oslostack/oslostack.yml
```

```yml
glance_backend_ceph: "yes"
ceph_glance_user: "glance"

cinder_backend_ceph: "yes"
ceph_cinder_user: "cinder"

nova_backend_ceph: "yes"
ceph_nova_user: "nova"

cinder_hdd_backend_pool: "hdd-volumes"
cinder_hdd_backend_name: "hdd"
cinder_ssd_backend_pool: "ssd-volumes"
cinder_ssd_backend_name: "ssd"
```

### 执行部署命令

```shell
kolla-ansible -i ./inventory -e @/opt/oslostack/oslostack.yml bootstrap-servers
kolla-ansible -i ./inventory -e @/opt/oslostack/oslostack.yml prechecks
kolla-ansible -i ./inventory -e @/opt/oslostack/oslostack.yml deploy
kolla-ansible -i ./inventory -e @/opt/oslostack/oslostack.yml post-deploy
```

### 使用 OpenStack

安装 openstack 命令行

```shell
pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/wallaby
```

加载 openstack 凭证文件

```shell
. /etc/kolla/admin-openrc.sh
```

执行 openstack 命令

```shell
openstack compute service list
```

获取 admin 用户密码

```shell
cat /etc/kolla/passwords.yml | grep keystone_admin_password | awk '{print $2}'
```
