# 快速开始

本指南提供了使用 ceph-ansible 与 kolla-ansible 在裸金属服务器或虚拟机上部署 Ceph 与 OpenStack 的逐步说明。

- ceph 版本：pacific
- openstack 版本：yoga

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

可以使用镜像源加速软件包下载速度。
对于 CentOS 8 Stream，使用以下命令替换默认的配置。

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

#### 安装 openstack 软件仓库

```shell
dnf install -y centos-release-openstack-yoga
```

#### 网络配置

##### 安装 openvswitch

```shell
dnf install -y openvswitch
systemctl start openvswitch && systemctl enable openvswitch
```

##### 配置 IPv4 forwarding

```shell
vi /etc/sysctl.conf
net.ipv4.ip_forward = 1
sysctl -p /etc/sysctl.conf
```

##### 使用 network-scripts 配置网络

```shell
# 卸载NetworkManager
dnf remove -y NetworkManager

# 禁用firewalld
systemctl stop firewalld && systemctl disable firewalld

# 配置网桥
vim /etc/sysconfig/network-scripts/ifcfg-br-ex
DEVICE=br-ex
ONBOOT=yes
HOTPLUG=no
NM_CONTROLLED=no
PEERDNS=no
DEVICETYPE=ovs
TYPE=OVSBridge
MTU=1500
OVS_EXTRA="set bridge br-ex fail_mode=standalone -- del-controller br-ex"

# 配置bond
vim /etc/sysconfig/network-scripts/ifcfg-bond0
DEVICE=bond0
ONBOOT=yes
HOTPLUG=no
NM_CONTROLLED=no
PEERDNS=no
DEVICETYPE=ovs
TYPE=OVSPort
OVS_BRIDGE=br-ex
BONDING_OPTS="mode=4 miimon=100"
MTU=1500

# 配置 bond 从属网卡
vim /etc/sysconfig/network-scripts/ifcfg-eno1
DEVICE=eno1
ONBOOT=yes
HOTPLUG=no
NM_CONTROLLED=no
PEERDNS=no
MASTER=bond0
SLAVE=yes
BOOTPROTO=none
MTU=1500

vim /etc/sysconfig/network-scripts/ifcfg-eno2
DEVICE=eno2
ONBOOT=yes
HOTPLUG=no
NM_CONTROLLED=no
PEERDNS=no
MASTER=bond0
SLAVE=yes
BOOTPROTO=none
MTU=1500

# 配置ovs子接口
vim /etc/sysconfig/network-scripts/ifcfg-vlan100
DEVICE=vlan100
ONBOOT=yes
HOTPLUG=no
NM_CONTROLLED=no
PEERDNS=no
DEVICETYPE=ovs
TYPE=OVSIntPort
OVS_BRIDGE=br-ex
OVS_OPTIONS="tag=100"
MTU=1500
BOOTPROTO=static
IPADDR=10.0.10.11
NETMASK=255.255.255.0

systemctl restart network
```

##### 配置 DNS

```shell
vi /etc/resolv.conf
nameserver 119.29.29.29
```

##### 配置时钟同步

```shell
dnf install -y chrony
systemctl start chronyd && systemctl enable chronyd
chronyc sources
```

##### 检查并清理硬盘

```shell
dd if=/dev/zero of=/dev/sdb count=10 bs=1M
```

## 在部署节点上进行

### 安装发行版的基础依赖包

```shell
dnf install -y git vim python3 sshpass tmux python3-devel libffi-devel gcc openssl-devel python3-libselinux python3-netaddr
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
git clone -b stable-6.0 https://github.com/ceph/ceph-ansible.git
git clone -b stable/yoga https://opendev.org/openstack/kolla-ansible.git
```

### 准备 python 虚拟环境

```shell
python3 -m venv /opt/oslostack/venv
source /opt/oslostack/venv/bin/activate
```

### 安装 python 依赖

```shell
pip install -U pip 
pip install 'ansible>=4,<6'
```

### 安装 kolla-ansible

```shell
cd ./kolla-ansible
git checkout stable/yoga
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

### 安装 kolla-ansible ansible galaxy 依赖项

```shell
kolla-ansible install-deps
```

### 修改 ansible 配置文件

```shell
mkdir -p /etc/ansible
vim /etc/ansible/ansible.cfg
[defaults]
host_key_checking=False
pipelining=True
forks=100
log_path = /var/log/oslostack-ansible.log

[privilege_escalation]
become = True
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

[grafana-server:children]
mons
```

### 测试节点连通性

```shell
ansible -i ./inventory all -m ping
```

### 填写 kolla-ansible 配置

```shell
cd /opt/oslostack
vim oslostack.yml
```

```yml
# 离线 registry
# docker_registry: "10.0.10.11:4000"
# docker_registry_insecure: "yes"
kolla_base_distro: "centos"
kolla_install_type: "source"

# 根据实际情况填写
network_interface: "vlan96"
neutron_external_interface: "bond0"
kolla_internal_vip_address: "10.1.0.250"
enable_cinder: "yes"
enable_cinder_backup: "no"
enable_fluentd: "no"
enable_openvswitch: "no"
enable_heat: "no"
```

### 生成 kolla 密码

```shell
kolla-genpwd
```

### 节点初始化

```shell
# 添加占位符
vim /etc/kolla/globals.yml
---
dummy:

kolla-ansible -i ./inventory -e @/opt/oslostack/oslostack.yml bootstrap-servers -vv
```

### 准备离线 registry（可选）

#### 启动 docker registry

```shell
docker pull registry:2
docker run -d -p 4000:5000 --restart always --name registry registry:2
```

#### ceph 容器镜像

非容器化部署情况下无需准备 ceph 容器镜像

```shell
docker pull quay.io/ceph/daemon:latest-pacific
docker tag quay.io/ceph/daemon:latest-pacific 10.0.10.11:4000/ceph/daemon:latest-pacific
docker push 10.0.10.11:4000/ceph/daemon:latest-pacific
```

#### openstack 容器镜像

```shell
# 以 nova-api 示例
docker pull kolla/centos-source-nova-api:yoga
docker tag kolla/centos-source-nova-api:yoga 10.0.10.11:4000/kolla/centos-source-nova-api:yoga
docker push 10.0.10.11:4000/kolla/centos-source-nova-api:yoga

# 拉取所有镜像
kolla-ansible -i ./inventory -e @/opt/oslostack/oslostack.yml pull -vv
...
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

- stable-5.0 支持 Ceph pacific 版本。这个分支需要 Ansible 2.9 版本。

- stable-6.0 支持 Ceph pacific 版本。这个分支需要 Ansible 2.9 版本。

- stable-master 支持 Ceph master 分支 版本。这个分支需要 Ansible 2.10 版本。

#### 切换到您希望使用的某个版本分支

```shell
cd ceph-ansible
git checkout stable-6.0
```

#### 准备 ceph-ansible python 虚拟环境

```shell
deactivate
python3 -m venv /opt/oslostack/ceph-venv
source /opt/oslostack/ceph-venv/bin/activate
pip install -U pip
pip install -r ./ceph-ansible/requirements.txt
```

#### 安装 ceph-ansible ansible galaxy 依赖项

```shell
cd ceph-ansible
ansible-galaxy install -r requirements.yml
```

### 填写 ceph-ansible 配置

```shell
cd /opt/oslostack
vim oslostack.yml
```

```yml
# ceph
cephx: true
# 离线 registry
# ceph_docker_registry: "10.0.10.11:4000"
# containerized_deployment: true
# container_binary: docker
# container_package_name: docker-ce
ceph_origin: distro
configure_firewall: false

osd_scenario: lvm
osd_objectstore: bluestore
osd_auto_discovery: true
# 显式设置 db 空间大小，单位为 bytes，默认 -1 为平分空间
block_db_size: -1

# 根据实际情况填写
public_network: "192.168.3.0/24"
cluster_network: "192.168.3.0/24"
monitor_interface: vlan96

ntp_service_enabled: false

dashboard_admin_password: oslostack
grafana_admin_password: oslostack

crush_rule_config: true
crush_rule_hdd:
  name: hdd
  root: default
  type: host
  default: false
  class: hdd
# 如无 ssd 设备，请将 ssd 相关配置注释掉
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
# 如没有 ssd 设备，注释相关配置
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
```

### 执行 ceph-ansible 部署命令

```shell
cd /opt/oslostack/ceph-ansible
ansible-playbook -i /opt/oslostack/inventory site.yml.sample -e @/opt/oslostack/oslostack.yml -vv
```

## 使用 kolla-ansible 部署 OpenStack

### 填写 OpenStack 与 Ceph 对接配置

```shell
mkdir -p /etc/kolla/config/glance

cp /etc/ceph/ceph.client.glance.keyring /etc/kolla/config/glance/
cp /etc/ceph/ceph.conf /etc/kolla/config/glance/

mkdir -p /etc/kolla/config/cinder/cinder-volume

vim /etc/kolla/config/cinder/cinder-volume.conf
[DEFAULT]
enabled_backends=hdd,ssd

[hdd]
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = cinder
backend_host = rbd:hdd_volumes
rbd_pool = hdd-volumes
volume_backend_name = hdd
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_secret_uuid = XXX # /etc/kolla/passwords.yml

[ssd]
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = cinder
backend_host = rbd:ssd_volumes
rbd_pool = ssd-volumes
volume_backend_name = ssd
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_secret_uuid = XXX

cp /etc/ceph/ceph.client.cinder.keyring /etc/kolla/config/cinder/cinder-volume
cp /etc/ceph/ceph.conf /etc/kolla/config/cinder/

mkdir -p /etc/kolla/config/nova
cp /etc/ceph/ceph.client.cinder.keyring /etc/kolla/config/nova/
cp /etc/ceph/ceph.conf /etc/kolla/config/nova/

vim /opt/oslostack/oslostack.yml
```

```yml
glance_backend_ceph: "yes"
cinder_backend_ceph: "yes"
nova_backend_ceph: "yes"
```

### 调整可用的 VLAN ID 范围

```shell
mkdir /etc/kolla/config/neutron
vim /etc/kolla/config/neutron/ml2_conf.ini
[ml2_type_vlan]
network_vlan_ranges = physnet1:1:4094
```

### 执行部署命令

```shell
deactivate
source /opt/oslostack/venv/bin/activate
kolla-ansible -i ./inventory -e @/opt/oslostack/oslostack.yml prechecks -vv -e prechecks_enable_host_ntp_checks=false
kolla-ansible -i ./inventory -e @/opt/oslostack/oslostack.yml deploy -vv
kolla-ansible -i ./inventory -e @/opt/oslostack/oslostack.yml post-deploy -vv
```

### 使用 OpenStack

安装 openstack 命令行

```shell
pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/yoga
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

上传镜像

```shell
curl -OL https://cloud.centos.org/centos/8-stream/x86_64/images/CentOS-Stream-GenericCloud-8-20220125.1.x86_64.qcow2
openstack image create --disk-format qcow2 --container-format bare --public --property os_type=linux --file ./CentOS-Stream-GenericCloud-8-20220125.1.x86_64.qcow2 CentOS8-Stream --progress
```

创建卷类型
```shell
openstack volume type create hdd --property volume_backend_name=hdd --property RESKEY:availability_zones=nova
openstack volume type create ssd --property volume_backend_name=ssd --property RESKEY:availability_zones=nova
```
