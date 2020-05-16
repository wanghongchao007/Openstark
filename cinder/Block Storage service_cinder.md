 https://docs.openstack.org/cinder/train/install/ 

# 一、安装和配置控制节点

## 1.环境准备
- #### 在安装和配置块存储服务之前，必须创建数据库，服务凭证和API端点

### 1.创建数据库
- 使用数据库访问客户端以root用户身份连接到数据库服务器：

```shell
[root@controller ~]# mysql -u root -pCom.123
```
- 创建cinder数据库：

```shell
MariaDB [(none)]> CREATE DATABASE cinder;
Query OK, 1 row affected (0.000 sec)
```
- 授予对cinder数据库的适当访问权限：
```shell
MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'Com.123';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%'  IDENTIFIED BY 'Com.123';
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> exit
Bye
```
### 2.使用admin凭据来访问仅管理员CLI命令：
```shell
[root@controller ~]# . admin-openrc
```
###  3.创建服务凭证
- 创建一个cinder用户：

```shell
[root@controller ~]# openstack user create --domain default --password-prompt cinder
User Password: Com.123
Repeat User Password: Com.123
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 05a45b7f94c44d6a8de9a6bd1cf185c8 |
| name                | cinder                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```
- admin向cinder用户添加角色：

```shell
[root@controller ~]# openstack role add --project service --user cinder admin
```
- 创建cinderv2和cinderv3服务实体：
```shell
[root@controller ~]# openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
ler:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev2 internal http://controller:8776/v2/%\(project_id\)s+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Block Storage          |
| enabled     | True                             |
| id          | 9c3f877634f14fd58ef3301efa7a7520 |
| name        | cinderv2                         |
| type        | volumev2                         |
+-------------+----------------------------------+
[root@controller ~]# openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Block Storage          |
| enabled     | True                             |
| id          | 1f69c4ec5914455b853ba5d5aa70bc55 |
| name        | cinderv3                         |
| type        | volumev3                         |
+-------------+----------------------------------+
```
###  4.创建块存储服务API端点：
```shell

[root@controller ~]# openstack endpoint create --region RegionOne volumev2 public http://controller:8776/v2/%\(project_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | 8c30c958e6344e76b2c00214578d8d9a         |
| interface    | public                                   |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 9c3f877634f14fd58ef3301efa7a7520         |
| service_name | cinderv2                                 |
| service_type | volumev2                                 |
| url          | http://controller:8776/v2/%(project_id)s |
+--------------+------------------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne volumev2 internal http://controller:8776/v2/%\(project_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | eb8277ea3af94bdea4faa7f847e422fd         |
| interface    | internal                                 |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 9c3f877634f14fd58ef3301efa7a7520         |
| service_name | cinderv2                                 |
| service_type | volumev2                                 |
| url          | http://controller:8776/v2/%(project_id)s |
+--------------+------------------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne volumev2 admin http://controller:8776/v2/%\(project_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | 7d2658f4d12644539296a824879eec8c         |
| interface    | admin                                    |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 9c3f877634f14fd58ef3301efa7a7520         |
| service_name | cinderv2                                 |
| service_type | volumev2                                 |
| url          | http://controller:8776/v2/%(project_id)s |
+--------------+------------------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne volumev3 public http://controller:8776/v3/%\(project_id\)s

+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | ec6ba7546f27460bba1283c15b8a6796         |
| interface    | public                                   |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 1f69c4ec5914455b853ba5d5aa70bc55         |
| service_name | cinderv3                                 |
| service_type | volumev3                                 |
| url          | http://controller:8776/v3/%(project_id)s |
+--------------+------------------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne volumev3 internal http://controller:8776/v3/%\(project_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | 69299cc350f14b32b31101ead12f1eef         |
| interface    | internal                                 |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 1f69c4ec5914455b853ba5d5aa70bc55         |
| service_name | cinderv3                                 |
| service_type | volumev3                                 |
| url          | http://controller:8776/v3/%(project_id)s |
+--------------+------------------------------------------+
[root@controller ~]# openstack endpoint create --region  RegionOne volumev3 admin http://controller:8776/v3/%\(project_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | dc40d08595dc46c684619c191ebce95b         |
| interface    | admin                                    |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 1f69c4ec5914455b853ba5d5aa70bc55         |
| service_name | cinderv3                                 |
| service_type | volumev3                                 |
| url          | http://controller:8776/v3/%(project_id)s |
+--------------+------------------------------------------+

```
##  2.安装和配置的部件
### 1.安装软件包：
```shell
[root@controller ~]# yum install openstack-cinder -y
```
###  2.编辑**/etc/cinder/cinder.conf**文件并完成以下操作：
- 配置数据库访问`[database]`
- 配置RabbitMQ 消息队列访问：`[DEFAULT]`
- 配置身份服务访问：`[keystone_authtoken]``[DEFAULT]`
- 配置my_ip选项以使用控制器节点的管理接口IP地址`[DEFAULT]`
- 配置锁定路径：[oslo_concurrency]
```shell
[database]
connection = mysql+pymysql://cinder:Com.123@controller/cinder
[DEFAULT]
transport_url = rabbit://openstack:Com.123@controller
auth_strategy = keystone
my_ip = 10.0.0.20

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = Com.123
[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
```
### 3.填充块存储数据库：
-  忽略此输出中的所有弃用消息。 

```shell
[root@controller ~]# su -s /bin/sh -c "cinder-manage db sync" cinder
Deprecated: Option "logdir" from group "DEFAULT" is deprecated. Use option "log-dir" from group "DEFAULT".
```
##  3.配置计算以使用块存储
###  3.1.编辑**/etc/nova/nova.conf**文件并向其中添加以下内容：
```shell
[cinder]
os_region_name = RegionOne
```
##  4.完成安装
###  4.1.重新启动Compute API服务：
```shell
[root@controller ~]# systemctl restart openstack-nova-api.service
```
###  4.2.启动块存储服务，并将其配置为在系统启动时启动：
```shell
[root@controller ~]# systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service
[root@controller ~]# systemctl start  openstack-cinder-api.service openstack-cinder-scheduler.service
```

# 二、安装和配置存储节点
## 环境准备
### 1.安装支持的实用程序包：
- 安装LVM软件包：
- 启动LVM元数据服务，并将其配置为在系统引导时启动：
```shell
[root@block ~]# yum install lvm2 device-mapper-persistent-data  -y
[root@block ~]# systemctl enable lvm2-lvmetad.service
[root@block ~]# systemctl start lvm2-lvmetad.service
```
### 2.创建LVM物理卷/dev/sdb：

```shell
[root@block ~]# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
```

### 3.创建LVM卷组cinder-volumes：
```shell
[root@block ~]# vgcreate cinder-volumes /dev/sdb
  Volume group "cinder-volumes" successfully created
```
### 4.只有实例可以访问块存储卷。但是，底层操作系统管理与卷关联的设备。默认情况下，LVM卷扫描工具会在/dev目录中扫描 包含卷的块存储设备。如果项目在其卷上使用LVM，则扫描工具会检测到这些卷并尝试对其进行缓存，这可能导致基础操作系统卷和项目卷出现各种问题。您必须将LVM重新配置为仅扫描包含cinder-volumes卷组的设备。编辑 **/etc/lvm/lvm.conf**文件并完成以下操作：
- 在该devices部分中，添加一个接受/dev/sdb设备并拒绝所有其他设备的过滤 器
- 滤波器阵列中的每个项目开始于a用于接受或 r用于拒绝，并且包括用于所述装置名称的正则表达式。该阵列必须r/.*/以拒绝任何剩余的设备结尾。您可以使用vgs -vvvv命令测试过滤器。
  - 如果存储节点在操作系统磁盘上使用LVM，则还必须将关联的设备添加到过滤器中。例如，如果/dev/sda设备包含操作系统：
  - 同样，如果您的计算节点在操作系统磁盘上使用LVM，则还必须/etc/lvm/lvm.conf在这些节点上的文件中修改过滤器， 使其仅包括操作系统磁盘。例如，如果/dev/sda 设备包含操作系统：
  ```shell
  filter = [ "a/sda/", "a/sdb/", "r/.*/"]
  filter = [ "a/sda/", "r/.*/"]
  ```
```shell
因为我是储存节点所以我添加以下内容
[root@block ~]# vim /etc/lvm/lvm.conf
devices {
....
filter = [ "a/sda/", "a/sdb/", "r/.*/"]
```
## 安装和配置的部件
### 1.安装软件包：

```shell
[root@block ~]# yum install openstack-cinder targetcli python-keystone -y
```

### 2.编辑**/etc/cinder/cinder.conf**文件并完成以下操作：
- 配置数据库访问：`[database]`
- 配置身份服务访问：`[DEFAULT]``[keystone_authtoken]`
- 配置my_ip选项：`[DEFAULT]`MANAGEMENT_INTERFACE_IP_ADDRESS为存储节点上管理网络接口的IP地址
- 在该[lvm]部分中，为LVM后端配置LVM驱动程序，cinder-volumes卷组，iSCSI协议和适当的iSCSI服务。如果该[lvm]部分不存在，请创建它
- 启用LVM后端：`[DEFAULT]`
- 配置图像服务API的位置：`[DEFAULT]`
- 配置锁定路径：`[oslo_concurrency]`

```shell
[database]
connection = mysql+pymysql://cinder:Com.123@controller/cinder


[DEFAULT]
glance_api_servers = http://controller:9292
enabled_backends = lvm
my_ip = 10.0.0.22
auth_strategy = keystone
transport_url = rabbit://openstack:Com.123@controller

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = Com.123

[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
target_protocol = iscsi
target_helper = lioadm

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
```
## 启动块存储卷服务及其相关性，并将其配置为在系统启动时启动：
```shell
[root@block ~]# systemctl enable openstack-cinder-volume.service target.service
[root@block ~]# systemctl start openstack-cinder-volume.service target.service
```
# 三、安装和配置备份服务
## 1.安装软件包：
```shell
[root@block-backup ~]# yum install openstack-cinder  -y
```
## 2.编辑**/etc/cinder/cinder.conf**文件并完成以下操作：
- 配置备份选项：`[DEFAULT]`
- 替换SWIFT_URL为对象存储服务的URL。可以通过显示对象库API端点来找到URL
```shell
[root@block-backup ~]# vim /etc/cinder/cinder.conf
[DEFAULT]
backup_driver = cinder.backup.drivers.swift.SwiftBackupDriver
backup_swift_url = SWIFT_URL
[root@block-backup ~]# openstack catalog show object-store
```
## 3.启动块存储备份服务，并将其配置为在系统启动时启动：
```shell
[root@block-backup ~]# systemctl enable openstack-cinder-backup.service
[root@block-backup ~]# systemctl start openstack-cinder-backup.service
```
# 四、验证cider服务
## 1.来源admin凭据来访问仅管理员CLI命令：
```shell
source  admin-openrc
```
## 2.列出服务组件以验证每个进程是否成功启动：
```shell
[root@controller ~]# openstack volume service list
+------------------+------------+------+---------+-------+----------------------------+
| Binary           | Host       | Zone | Status  | State | Updated At                 |
+------------------+------------+------+---------+-------+----------------------------+
| cinder-scheduler | controller | nova | enabled | up    | 2020-05-16T13:36:34.000000 |
| cinder-volume    | block@lvm  | nova | enabled | up    | 2020-05-16T13:36:35.000000 |
+------------------+------------+------+---------+-------+----------------------------+

```