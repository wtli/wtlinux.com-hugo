---
title: "Ceph快速入门"
date: 2018-08-23T14:00:56+08:00
draft: false
---
# 安装ceph-deploy

## 预安装

### Ceph-deploy设置
- ceph-deploy工具在管理节点上的目录之外运行。任何具有网络连接和现代python环境和ssh（如Linux）的主机都应该可以工作

架构图如下：

![](https://ws4.sinaimg.cn/large/006tNbRwgy1fuggjt3mc6j30fc07tt90.jpg)

CentOS7：

```
//注册秘钥
sudo subscription-manager repos --enable=rhel-7-server-extras-rpms  
//安装并注册EPEL存储库
sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm 
// 来添加yum仓库文件
cat << EOM > /etc/yum.repos.d/ceph.repo
[ceph-noarch]
name=Ceph noarch packages
baseurl=https://download.ceph.com/rpm-{ceph-stable-release}/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
EOM
//更新仓库并安装ceph-deploy
sudo yum update
sudo yum install ceph-deploy

```

### Ceph节点设置

- 管理节点必须具有对Ceph节点的无密码SSH访问。当ceph-deploy以用户身份登录到Ceph节点时，该特定用户必须具有无密码的sudo权限。


CentOS7：

#### 安装时间服务器
- 建议安装时间服务器，特别是Ceph Monitor节点，以免时间不一致导致问题


```shell
sudo yum install ntp ntpdate ntp-doc
```
#### 安装SSH服务器
- 确保SSH服务器在所有Ceph节点上运行

```shell
sudo yum install openssh-server
```

#### 创建Ceph部署用户
- ceph-deploy实用程序必须以具有无密码sudo权限的用户身份登录到Ceph节点，因为它需要安装软件和配置文件而不提示输入密码。
- 最新版本的ceph-deploy支持--username选项，因此您可以指定任何具有无密码sudo的用户（包括root用户，但不建议这样做）。 
- 我们建议在群集中的所有Ceph节点上为ceph-deploy创建特定用户。 请不要使用“ceph”作为用户名。

```shell
//在每个Ceph节点上创建一个新用户。
ssh user@ceph-server
sudo useradd -d /home/{username} -m {username}
sudo passwd {username}

//对于添加到每个Ceph节点的新用户，请确保该用户具有sudo权限。
echo "{username} ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/{username}
sudo chmod 0440 /etc/sudoers.d/{username}
```


#### 启动无秘钥登陆
- 由于ceph-deploy不会提示输入密码，因此必须在admin节点上生成SSH密钥并将公钥分发给每个Ceph节点。 ceph-deploy将尝试为初始监视器生成SSH密钥。

```shell
//生成SSH密钥，但不要使用sudo或root用户。将密码保留为空
ssh-keygen

Generating public/private key pair.
Enter file in which to save the key (/ceph-admin/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /ceph-admin/.ssh/id_rsa.
Your public key has been saved in /ceph-admin/.ssh/id_rsa.pub.
//将密钥复制到每个Ceph节点，将{username}替换为您使用Create a Ceph Deploy User创建的用户名。
ssh-copy-id {username}@node1
ssh-copy-id {username}@node2
ssh-copy-id {username}@node3
//推荐修改ceph-deploy管理节点的〜/.ssh/config文件，以便ceph-deploy可以以您创建的用户身份登录到Ceph节点
//而无需在每次执行时指定--username {username}部署。
//这有一个额外的好处，即简化ssh和scp的使用。 将{username}替换为您创建的用户名：
Host node1
   Hostname node1
   User {username}
Host node2
   Hostname node2
   User {username}
Host node3
   Hostname node3
   User {username}
```
#### 打开网络

编辑/etc/sysconfig/network-scripts/ifcfg-eth0修改ONBOOT=yes

#### 确保连通性
在/etc/hosts中添加各主机名和IP地址的解析

#### 开放端口
Ceph Monitor默认端口6789

Ceph OSDs默认端口在6800-7300端口之间通信

#### 关闭防火墙
```shell
systemctl stop firewalld
systemctl disable firewalld
```
#### 更精准配置防火墙
```shell
//Ceph Mon节点
sudo firewall-cmd --zone=public --add-service=ceph-mon --permanent
//OSDs and MDSs
sudo firewall-cmd --zone=public --add-service=ceph --permanent
//让配置生效
sudo firewall-cmd --reload
//防火墙规则
sudo iptables -A INPUT -i {iface} -p tcp -s {ip-address}/{netmask} --dport 6789 -j ACCEPT
//保存配置
/sbin/service iptables save

```
####  tty
使用visudo配置sudo权限

#### selinux
```shell
sudo setenforce 0
```

#### 依赖包
```shell
sudo yum install yum-plugin-priorities
sudo yum install yum-plugin-priorities --enablerepo=rhel-7-server-optional-rpms
```



### 概要
这样就完成了安装前配置了。下一步，开始集群部署。


## 存储集群

搭建此集群之前，请先完成初始化设置。

集群拓扑图如下：

![](https://ws1.sinaimg.cn/large/006tNbRwgy1fuigvj38fyj30e807kaac.jpg)

我们将在管理节点上使用ceph-deploy启动Ceph存储集群。创建一个三个Ceph节点集群，以便您可以探索Ceph功能。

第一步：创建一个带有一个Ceph Monitor和三个Ceph OSD守护进程的Ceph存储集群。

第二步：一旦群集达到活动+干净状态，通过添加第四个Ceph OSD，一个元数据服务器和另外两个Ceph监视器守护进程来扩展它。

第三步：为获得最佳结果，请在管理节点上创建一个目录，以维护ceph-deploy为您的集群生成的配置文件和密钥。

```shell
mkdir my-cluster
cd my-cluster
```
ceph-deploy实用程序将文件输出到当前目录。执行ceph-deploy时，请确保您位于此目录中。

<font color=red>注意:如果您以其他用户身份登录，请不要使用sudo调用ceph-deploy或以root身份运行它，因为它不会发出远程主机上所需的sudo命令。</font>
### 重置服务器

- 如果您遇到麻烦并且想重新开始，请执行以下操作以清除Ceph软件包，并清除其所有数据和配置：


```shell
ceph-deploy purge {ceph-node} [{ceph-node}]
ceph-deploy purgedata {ceph-node} [{ceph-node}]
ceph-deploy forgetkeys
rm ceph.*

```

- 如果执行purge，则必须重新安装Ceph。最后一个rm命令删除在先前安装期间由本地ceph-deploy写出的所有文件。

### 创建一个集群
- 在您为保存配置详细信息而创建的目录的管理节点上，使用ceph-deploy执行以下步骤。

#### 创建集群
```shell
ceph-deploy new {initial-monitor-node(s)}
例：
ceph-deploy new node1
```

#### 配置网络

- 如果您有多个网络接口，请在Ceph配置文件的[global]部分下添加公共网络设置。有关详细信息，请参阅[网络配置](http://docs.ceph.com/docs/master/rados/configuration/network-config-ref)参考

```shell
public network = {ip-address}/{bits}
例：
public network = 10.1.2.0/24

```

#### IPv6配置
```shell
echo ms bind ipv6 = true >> ceph.conf
```

#### 安装Ceph包
```shell
ceph-deploy install {ceph-node} [...]
例:
ceph-deploy install node1 node2 node3
```
- ceph-deploy实用程序将在每个节点上安装Ceph。


#### 部署初始监视器并收集密钥
```shell
ceph-deploy mon create-initial

```
完成此过程后，您的本地目录应具有以下密钥环：

```shell
ceph.client.admin.keyring
ceph.bootstrap-mgr.keyring
ceph.bootstrap-osd.keyring
ceph.bootstrap-mds.keyring
ceph.bootstrap-rgw.keyring
ceph.bootstrap-rbd.keyring
```
<font color=red>注意:如果此过程失败并显示类似于“无法找到/etc/ceph/ceph.client.admin.keyring”的消息，请确保ceph.conf中为监视节点列出的IP是公网IP，而不是私有IP。</font>

#### 优化deploy节点
- 使用ceph-deploy将配置文件和管理密钥复制到您的管理节点和Ceph节点，这样您就可以使用ceph CLI而无需在每次执行命令时指定监视器地址和ceph.client.admin.keyring。

```shell
ceph-deploy admin {ceph-node(s)}
例：
ceph-deploy admin node1 node2 node3


```

#### 部署管理节点
```shell
ceph-deploy mgr create node1  *Required only for luminous+ builds, i.e >= 12.x builds*
```

#### 添加三个OSD节点
- 出于这些说明的目的，我们假设您在每个节点中都有一个名为/ dev / vdb的未使用磁盘。确保设备当前未使用且不包含任何重要数据。


```shell
ceph-deploy osd create –data {device} {ceph-node}

例：
ceph-deploy osd create --data /dev/vdb node1
ceph-deploy osd create --data /dev/vdb node2
ceph-deploy osd create --data /dev/vdb node3
```

#### 检查集群健康
```shell
//如果健康会返回OK
ssh node1 sudo ceph health
//更详细的集群状态
ssh node1 sudo ceph -s
```

至此，如果健康会返回OK，Ceph集群创建完成。
### 集群扩展

- 启动并运行基本群集后，下一步是扩展群集。将Ceph元数据服务器添加到node1。然后将Ceph Monitor和Ceph Manager添加到node2和node3，以提高可靠性和可用性。

架构拓扑图如下：

![](https://ws1.sinaimg.cn/large/006tNbRwgy1fuijq5gutsj30fi0b4dg9.jpg)
#### 添加元数据服务器

- 要使用CephFS，您至少需要一个元数据服务器。执行以下操作以创建元数据服务器：

```shell
ceph-deploy mds create {ceph-node}
例：
ceph-deploy mds create node1
```

#### 添加监视器

- Ceph存储集群需要至少运行一个Ceph Monitor和Ceph Manager。
- 为了实现高可用性，Ceph存储集群通常运行多个Ceph监视器，因此单个Ceph监视器的故障不会导致Ceph存储集群崩溃。
- Ceph使用Paxos算法，该算法需要大多数监视器（即比N/2更多，其中N是监视器的数量）才能形成法定人数。虽然这不是必需的，但监视器的数量往往越多更好。


```shell
//将两个Ceph监视器添加到您的群集：
ceph-deploy mon add {ceph-nodes}
例：
ceph-deploy mon add node2 node3
```

- 一旦你添加了新的Ceph监视器，Ceph将开始同步监视器并形成一个法定人数。您可以通过执行以下操作来检查仲裁状态：

```shell
ceph quorum_status --format json-pretty

```
<font color=red>提示：当您使用多个监视器运行Ceph时，您应该在每个监视器主机上安装和配置NTP。确保监视器时间是同步的。</font>
#### 添加管理器
- Ceph Manager守护进程以主/备用模式运行。部署其他管理器守护程序可确保在一个守护程序或主机发生故障时，另一个守护程序或主机可以在不中断服务的情况下进行接管。


```shell
//部署其他管理器守护程序
ceph-deploy mgr create node2 node3
//查看备用管理器
ssh node1 sudo ceph -s

```
#### 添加RGW实例
- 要使用Ceph的Ceph对象网关组件，必须部署RGW实例。执行以下命令以创建RGW的新实例：


```shell
ceph-deploy rgw create {gateway-node}
例：
ceph-deploy rgw create node1

```

- 默认情况下，RGW实例将侦听端口7480.可以通过在运行RGW的节点上编辑ceph.conf来更改此设置，如下所示：


```shell
[client]
rgw frontends = civetweb port=80
```

- 要使用IPv6地址，请使用：

```shell
[client]
rgw frontends = civetweb port=[::]:80
```

---------
### 存储/检索对象数据
- 要将对象数据存储在Ceph存储集群中，Ceph客户端必须：
	
	1.设置对象名称
	
	2.指定一个[pool](http://docs.ceph.com/docs/master/rados/operations/pools)

```shell
//查找对象位置
ceph osd map {poolname} {object-name}

```

#### 练习找到一个对象
- 作为练习，让我们创建一个对象。在命令行上使用rados put命令指定对象名称，包含某些对象数据的测试文件的路径和池名称。例如：

```shell
echo {Test-data} > testfile.txt
ceph osd pool create mytest 8
rados put {object-name} {file-path} --pool=mytest
rados put test-object-1 testfile.txt --pool=mytest
```

- 要验证Ceph存储集群是否存储了该对象，请执行以下命令：

```shell
rados -p mytest ls
```

- 确定对象位置


```shell
ceph osd map {pool-name} {object-name}
ceph osd map mytest test-object-1
```

- Ceph应该输出对象的位置。例如

```shell
osdmap e537 pool 'mytest' (1) object 'test-object-1' -> pg 1.d1743484 (1.4) -> up [1,0] acting [1,0]

```

- 要删除测试对象，只需使用rados rm命令将其删除即可。

```shell
rados rm test-object-1 --pool=mytest
```

- 要删除mytest pool


```shell
ceph osd pool rm mytest

```
（出于安全原因，您需要根据提示提供其他参数;删除pool会破坏数据。）

随着集群的发展，对象位置可能会动态变化。 Ceph动态重新平衡的一个好处是Ceph使您不必手动执行数据迁移或平移。

## CEPH客户端

### 块设备快速启动
- 块设备快速启动之前，您必须先执行“存储群集快速入门”中的步骤。在使用Ceph块设备之前，请确保您的Ceph存储群集处于活动+干净状态。
- Ceph块设备也称为RBD或RADOS块设备。

拓扑图如下：

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fuimcnau0ij30ez03bmx8.jpg)

- 您可以为您的ceph-client节点使用虚拟机，但不要在与Ceph存储群集节点相同的物理节点上执行以下过程（除非您使用VM）。


#### 安装Ceph

- 验证您是否具有适当版本的Linux内核。


```shell
lsb_release -a
uname -r
```

- 在admin节点上，使用ceph-deploy在ceph-client节点上安装Ceph。


```shell
ceph-deploy install ceph-client

```

- 在admin节点上，使用ceph-deploy将Ceph配置文件和ceph.client.admin.keyring复制到ceph-client。


```shell
ceph-deploy admin ceph-client

```
ceph-deploy实用程序将密钥环复制到/etc/ceph目录。确保密钥环文件具有适当的读取权限（例如，sudo chmod  +r /etc/ceph/ceph.client.admin.keyring）。

#### 创建块设备pool
- 在管理节点上，使用ceph工具创建pool（我们建议使用名称'rbd'）。
- 在管理节点上，使用rbd工具初始化池以供RBD使用：


```shell
rbd pool init <pool-name>

```
#### 配置块设备

```shell
//在ceph-client节点上，创建块设备映像。
rbd create foo --size 4096 --image-feature layering [-m {mon-IP}] [-k /path/to/ceph.client.admin.keyring]
//在ceph-client节点上，将图像映射到块设备。
sudo rbd map foo --name client.admin [-m {mon-IP}] [-k /path/to/ceph.client.admin.keyring]
//通过在ceph-client节点上创建文件系统来使用块设备。
sudo mkfs.ext4 -m0 /dev/rbd/rbd/foo
This may take a few moments.
//在ceph-client节点上挂载文件系统。
sudo mkdir /mnt/ceph-block-device
sudo mount /dev/rbd/rbd/foo /mnt/ceph-block-device
cd /mnt/ceph-block-device
```
选项：将块设备配置为自动映射并在引导时挂载（并在关闭时卸载/取消映射） - 请参阅[rbdmap](http://docs.ceph.com/docs/master/man/8/rbdmap)联机帮助页。

更多请参阅： [block devices](http://docs.ceph.com/docs/master/rbd)

### 文件系统快速入门

- 要使用CephFS快速入门指南，您必须首先执行“存储群集快速入门”指南中的过程。在管理主机上执行此快速入门。


#### 先决条件
- 验证您是否具有适当版本的Linux内核。有关详细信息，请参阅[OS建议](http://docs.ceph.com/docs/master/start/os-recommendations)


```shell
lsb_release -a
uname -r
```

- 在admin节点上，使用ceph-deploy在ceph-client节点上安装Ceph。

```shell
ceph-deploy install ceph-client
```

- 确保Ceph存储群集正在运行并处于活动+干净状态。此外，请确保至少运行一个Ceph元数据服务器。

```shell
ceph -s [-m {monitor-ip-address}] [-k {path/to/ceph.client.admin.keyring}]
```



#### 创建一个文件系统

- 您已经创建了MDS[存储群集快速入门](#存储集群)，但在创建一些pool和文件系统之前它不会变为活动状态。请参阅[创建Ceph文件系统](http://docs.ceph.com/docs/master/cephfs/createfs/)。

```shell
ceph osd pool create cephfs_data <pg_num>
ceph osd pool create cephfs_metadata <pg_num>
ceph fs new <fs_name> cephfs_metadata cephfs_data
```


#### 创建一个秘密文件
- Ceph存储集群在默认情况下启用身份验证的情况下运行。您应该有一个包含密钥的文件（即，不是密钥环本身）。要获取特定用户的密钥，请执行以下过程：

```shell
//在密钥环文件中标识用户的密钥。例如:
cat ceph.client.admin.keyring
//复制将使用已安装的CephFS文件系统的用户的密钥。它应该看起来像这样：
[client.admin]
   key = AQCj2YpRiAe6CxAA7/ETt7Hcl9IyxyYciVs47w==
//打开文本编辑器
//将密钥粘贴到空文件中。它应该看起来像这样：
AQCj2YpRiAe6CxAA7/ETt7Hcl9IyxyYciVs47w==
//使用用户名作为属性保存文件（例如，admin.secret）。
//确保文件权限适合用户，但对其他用户不可见。
```

#### 内核驱动程序
- 将CephFS挂载为内核驱动程序

```shell
sudo mkdir /mnt/mycephfs
sudo mount -t ceph {ip-address-of-monitor}:6789:/ /mnt/mycephfs

```

- Ceph存储集群默认使用身份验证。在“创建机密文件”部分中指定您创建的用户名和机密文件。例如：

```shell
sudo mount -t ceph 192.168.0.1:6789:/ /mnt/mycephfs -o name=admin,secretfile=admin.secret
```
<font color=red>注意在管理节点上安装CephFS文件系统，而不是服务器节点。</font>
#### 用户空间的文件系统（FUSE）

- 将CephFS挂载为用户空间（FUSE）中的文件系统。

```shell
sudo mkdir ~/mycephfs
sudo ceph-fuse -m {ip-address-of-monitor}:6789 ~/mycephfs
```

- Ceph存储集群默认使用身份验证。如果密钥环不在默认位置（即/ etc / ceph），请指定密钥环：

```shell
sudo ceph-fuse -k ./ceph.client.admin.keyring -m 192.168.0.1:6789 ~/mycephfs
```

#### 附加信息

有关其他信息，请参阅[CephFS](http://docs.ceph.com/docs/master/cephfs/)。 CephFS不如Ceph Block Device和Ceph Object Storage稳定。如果遇到问题，请参阅[故障排除](http://docs.ceph.com/docs/master/cephfs/troubleshooting)



### 对象存储快速入门

- 从firefly（v0.80）开始，Ceph Storage大大简化了Ceph对象网关的安装和配置。Gateway守护程序嵌入Civetweb，因此您无需安装Web服务器或配置FastCGI.此外，ceph-deploy可以安装网关包，生成密钥，配置数据目录并为您创建网关实例。

提示：Civetweb默认使用端口7480。您必须打开端口7480，或将端口设置为Ceph配置文件中的首选端口（例如，端口80）。

要启动Ceph对象网关，请按照以下步骤操作：


#### 安装Ceph对象网关
- 在客户端节点上执行预安装步骤。如果您打算使用Civetweb的默认端口7480，则必须使用firewall-cmd或iptables打开它。
- 从管理服务器的工作目录中，在客户机节点节点上安装Ceph对象网关包。例如：

```shell
ceph-deploy install --rgw <client-node> [<client-node> ...]

```


#### 创建Ceph对象网关实例

- 从管理服务器的工作目录中，在客户机节点上创建Ceph对象网关的实例。例如：


```shell
ceph-deploy rgw create <client-node>
```
网关运行后，您应该能够在端口7480上访问它。（例如，http//client-node:7480）。
#### 配置Ceph对象网关实例
- 要更改默认端口（例如，更改为端口80），请修改Ceph配置文件。 添加名为[client.rgw\<client-node>]的部分，将<client-node>替换为Ceph客户机节点的短节点名称（即hostname -s）。 例如，如果您的节点名称是client-node，请在[global]部分之后添加如下所示的部分：

```shell
[client.rgw.client-node]
rgw_frontends = "civetweb port=80"
```
>注意确保在rgw_frontends键/值对中的port = <port-number>之间不留空格。

>要点：如果您打算使用端口80，请确保Apache服务器未运行，否则将与Civetweb冲突。 我们建议在这种情况下删除Apache。

- 要使新端口设置生效，请重新启动Ceph对象网关。


```shell
// 在Red Hat Enterprise Linux 7和Fedora上，运行以下命令：
sudo systemctl restart ceph-radosgw.service
//在Red Hat Enterprise Linux 6和Ubuntu上，运行以下命令：
sudo service radosgw restart id=rgw.<short-hostname>
```

- 最后，检查以确保您选择的端口在节点的防火墙（例如，端口80）上打开。 如果未打开，请添加端口并重新加载防火墙配置。 例如：

```shell
sudo firewall-cmd --list-all
sudo firewall-cmd --zone=public --add-port 80/tcp --permanent
sudo firewall-cmd --reload
```

- 您应该能够进行未经身份验证的请求，并收到回复。 例如，没有这样的参数的请求：

```shell
http://<client-node>:80
```

- 应该得到这样的回应：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
  <Owner>
    <ID>anonymous</ID>
    <DisplayName></DisplayName>
  </Owner>
    <Buckets>
  </Buckets>
</ListAllMyBucketsResult>
```

有关其他管理和API详细信息，请参阅[配置Ceph对象网关指南](http://docs.ceph.com/docs/master/radosgw/config-ref)。
