# 一.控制器节点安装计算服务
### 环境准备
- #### 在安装和配置计算服务之前，必须创建数据库、服务凭据和 API 终结点。
### 1.1.创建数据库
- 使用数据库访问客户端以用户身份连接到数据库服务器：root
```shell
[root@controller ~]# mysql -u root -pCom.123
```
- 创建数据库
```shell
MariaDB [(none)]> CREATE DATABASE nova_api;
Query OK, 1 row affected (0.000 sec)
MariaDB [(none)]> CREATE DATABASE nova;
Query OK, 1 row affected (0.000 sec)
MariaDB [(none)]> CREATE DATABASE nova_cell0;
Query OK, 1 row affected (0.000 sec)
```
- 授予对数据库的适当访问权限：
```shell
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost'   IDENTIFIED BY 'Com.123';
Query OK, 0 rows affected (0.000 sec)
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%'           IDENTIFIED BY 'Com.123';
Query OK, 0 rows affected (0.000 sec)
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost'       IDENTIFIED BY 'Com.123';
Query OK, 0 rows affected (0.000 sec)
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%'               IDENTIFIED BY 'Com.123';
Query OK, 0 rows affected (0.000 sec)
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'Com.123';
Query OK, 0 rows affected (0.000 sec)
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%'         IDENTIFIED BY 'Com.123';
Query OK, 0 rows affected (0.000 sec)
```
### 1.2.源凭据以访问仅管理员 CLI 命令：admin

```shell
[root@controller ~]# . admin-openrc
```

### 1.3.创建计算服务凭据：
- 创建用户：nova
- **password**=Com.123
```shell
[root@controller ~]# openstack user create --domain default --password-prompt nova
User Password: Com.123
Repeat User Password: Com.123
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 7ba8fbe5960746d9948f6bd88202b460 |
| name                | nova                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```
- 将角色添加到用户：admin nova
```shell
[root@controller ~]# openstack role add --project service --user nova admin
```
- 创建服务实体：nova
```shell
[root@controller ~]# openstack service create --name nova  --description "OpenStack Compute" compute
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | bf917a74f31f42e58af012546372746f |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+
```
### 1.4.创建计算 API 服务终结点：
```shell
[root@controller ~]# openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 99ce7668f8e546a3a941030d044adab9 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | bf917a74f31f42e58af012546372746f |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 4ae20cc06fff4c59bedcac3aea341ece |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | bf917a74f31f42e58af012546372746f |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 24807dba9f2842468c6c208b2405d9a1 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | bf917a74f31f42e58af012546372746f |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+
```
## 2.安装和配置计算节点
### 2.1.安装软件包：

```shell
[root@controller ~]# yum install openstack-nova-api openstack-nova-conductor openstack-nova-novncproxy openstack-nova-scheduler -y
```

### 2.2.编辑文件并完成以下操作：**/etc/nova/nova.conf**
- 启用计算和元数据 API：`` [DEFAULT] ``
- 配置消息队列访问： `[DEFAULT]``RabbitMQ` 
- 配置使用控制器节点的管理接口 IP 地址的选项： `[DEFAULT]``my_ip` 
- 启用对网络服务的支持：``[DEFAULT]``
```shell
[DEFAULT]
my_ip = 10.0.0.20
use_neutron = true
enabled_apis = osapi_compute,metadata
firewall_driver = nova.virt.firewall.NoopFirewallDriver
transport_url = rabbit://openstack:Com.123@controller:5672/
```
- 配置数据库访问： `[api_database]``[database]` 
```shell
[api_database]
connection = mysql+pymysql://nova:Com.123@controller/nova_api
[database]
connection = mysql+pymysql://nova:Com.123@controller/nova
```
- 配置标识服务访问：  `[api]``[keystone_authtoken]` 
```shell
[api]
auth_strategy = keystone
[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = Com.123
```
- VNC 代理配置为使用控制器节点的管理接口 IP 地址： ``[vnc]``
```shell
[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip
```
- 配置影像服务 API 的位置： ``[glance]``
```shell
[glance]
api_servers = http://controller:9292
```
- 配置锁定路径： ``[oslo_concurrency]``
```shell
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
```
- 配置对放置服务的访问： ``[placement]``
```shell
[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = Com.123
```
-  硬件加速``[libvirt]``
  
  - ```shell
    [root@controller ~]# egrep -c '(vmx|svm)' /proc/cpuinfo
    0
    ## 如果此命令返回值，则计算节点不支持硬件加速，必须配置为使用QEMU而不是KVM.
    ```
```shell
[libvirt]
virt_type = qemu
```

### 2.3.填充数据库：nova-api
```shell
[root@controller ~]# su -s /bin/sh -c "nova-manage api_db sync" nova
```
### 2.4.注册数据库：cell0
```shell
[root@controller ~]# su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
```
### 2.5.创建单元格：cell1
```shell
[root@controller ~]# su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
5ec5347e-91bf-4a00-a015-33de33418bb3
```
### 2.6.填充新的数据库：
```shell
[root@controller ~]# su -s /bin/sh -c "nova-manage db sync" nova
```
### 2.7.验证cell0和cell1 是否正确注册：
```shell
[root@controller ~]# su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
+-------+--------------------------------------+------------------------------------------+-------------------------------------------------+----------+
|  名称 |                 UUID                 |              Transport URL               |                    数据库连接                   | Disabled |
+-------+--------------------------------------+------------------------------------------+-------------------------------------------------+----------+
| cell0 | 00000000-0000-0000-0000-000000000000 |                  none:/                  | mysql+pymysql://nova:****@controller/nova_cell0 |  False   |
| cell1 | 5ec5347e-91bf-4a00-a015-33de33418bb3 | rabbit://openstack:****@controller:5672/ |    mysql+pymysql://nova:****@controller/nova    |  False   |
+-------+--------------------------------------+------------------------------------------+-------------------------------------------------+----------+
```
### 2.8.完成安装
```shell
[root@controller ~]# systemctl enable \
>     openstack-nova-api.service \
>     openstack-nova-scheduler.service \
>     openstack-nova-conductor.service \
>     openstack-nova-novncproxy.service
[root@controller ~]# systemctl start \
>     openstack-nova-api.service \
>     openstack-nova-scheduler.service \
>     openstack-nova-conductor.service \
>     openstack-nova-novncproxy.service
```

# 二. 计算机节点安装计算服务
# 1.安装和配置组件
```shell
[root@computer ~]# yum install openstack-nova-compute -y
```
## 1.1.编辑文件并完成以下操作：**/etc/nova/nova.conf**
- 启用计算和元数据 API：``[DEFAULT]``
- 配置消息队列访问：``[DEFAULT]``
-  配置选项：`[DEFAULT]``my_ip` 
-  启用对网络服务的支持：`[DEFAULT]` 
```shell
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:Com.123@controller
my_ip = 10.0.0.21
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver
```
- 配置标识服务访问：``[api]`[keystone_authtoken]``
```shell
[api]
auth_strategy = keystone
[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = Com.123
```

- 启用和配置远程控制台访问：`[vnc]` 
```shell
[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html
```
- 配置影像服务 API 的位置：``[glance]``
```shell
[glance]
api_servers = http://controller:9292
```
- 配置锁定路径：``[oslo_concurrency]``
```shell
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
```
- 配置放置 API：``[placement]``
```shell
[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = Com.123
```
## 1.2.完成安装
- 确定计算节点是否支持虚拟机的硬件加速：
  - 如果此命令返回的值为，则计算节点支持硬件加速，这通常不需要额外配置。一个或多个
  - 如果此命令返回值，则计算节点不支持硬件加速，必须配置为使用QEMU而不是KVM.zerolibvirt
```shell
[root@computer ~]# egrep -c '(vmx|svm)' /proc/cpuinfo
4
```
- 编辑文件中的部分，如下所示：``[libvirt]`` **/etc/nova/nova.conf**
```shell
[libvirt]
virt_type = qemu
```
-  启动计算服务（包括其依赖项），并将其配置为在系统启动时自动启动： 
```shell
[root@computer ~]# systemctl enable libvirtd.service openstack-nova-compute.service
[root@computer ~]# systemctl start libvirtd.service openstack-nova-compute.service
```
# 2. 将计算节点添加到单元数据库
### 2.1.源管理凭据以启用仅管理员 CLI 命令，然后确认数据库中存在计算主机：
```shell
[root@controller ~]# . admin-openrc
[root@controller ~]# openstack compute service list --service nova-compute
+----+--------------+----------+------+---------+-------+----------------------------+
| ID | Binary       | Host     | Zone | Status  | State | Updated At                 |
+----+--------------+----------+------+---------+-------+----------------------------+
|  7 | nova-compute | computer | nova | enabled | up    | 2020-05-18T11:39:11.000000 |
+----+--------------+----------+------+---------+-------+----------------------------+
```
### 2.2.发现计算主机：
```shell
[root@controller ~]# su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting computes from cell 'cell1': 5ec5347e-91bf-4a00-a015-33de33418bb3
Found 0 unmapped computes in cell: 5ec5347e-91bf-4a00-a015-33de33418bb3
```
# 3.注意
### 添加新计算节点时，必须在控制器节点上运行以注册这些新的计算节点。或者，您可以在 中设置适当的间隔：nova-manage cell_v2 discover_hosts/etc/nova/nova.conf

```shell
[scheduler]
discover_hosts_in_cells_interval = 300
```
# 4.验证操作
-  https://docs.openstack.org/nova/train/install/verify.html 
## 1.源凭据以访问仅管理员 CLI 命令：admin
```shell
[root@controller ~]# . admin-openrc
```
## 2.列出服务组件，以验证每个流程的成功启动和注册：
```shell
[root@controller ~]# openstack compute service list
+----+----------------+------------+----------+---------+-------+----------------------------+
| ID | Binary         | Host       | Zone     | Status  | State | Updated At                 |
+----+----------------+------------+----------+---------+-------+----------------------------+
|  4 | nova-conductor | controller | internal | enabled | up    | 2020-05-15T09:41:55.000000 |
|  6 | nova-scheduler | controller | internal | enabled | up    | 2020-05-15T09:41:55.000000 |
+----+----------------+------------+----------+---------+-------+----------------------------+
```
## 3.在标识服务中列出 API 终结点，以验证与标识服务的连接：
```shell
[root@controller ~]# openstack catalog list
+-----------+-----------+-----------------------------------------+
| Name      | Type      | Endpoints                               |
+-----------+-----------+-----------------------------------------+
| glance    | image     | RegionOne                               |
|           |           |   admin: http://controller:9292         |
|           |           | RegionOne                               |
|           |           |   public: http://controller:9292        |
|           |           | RegionOne                               |
|           |           |   internal: http://controller:9292      |
|           |           |                                         |
| keystone  | identity  | RegionOne                               |
|           |           |   public: http://controller:5000/v3/    |
|           |           | RegionOne                               |
|           |           |   admin: http://controller:5000/v3/     |
|           |           | RegionOne                               |
|           |           |   internal: http://controller:5000/v3/  |
|           |           |                                         |
| neutron   | network   | RegionOne                               |
|           |           |   admin: http://controller:9696         |
|           |           | RegionOne                               |
|           |           |   internal: http://controller:9696      |
|           |           | RegionOne                               |
|           |           |   public: http://controller:9696        |
|           |           |                                         |
| placement | placement | RegionOne                               |
|           |           |   admin: http://controller:8778         |
|           |           | RegionOne                               |
|           |           |   internal: http://controller:8778      |
|           |           | RegionOne                               |
|           |           |   public: http://controller:8778        |
|           |           |                                         |
| nova      | compute   | RegionOne                               |
|           |           |   admin: http://controller:8774/v2.1    |
|           |           | RegionOne                               |
|           |           |   internal: http://controller:8774/v2.1 |
|           |           | RegionOne                               |
|           |           |   public: http://controller:8774/v2.1   |
|           |           |                                         |
+-----------+-----------+-----------------------------------------+
```
## 4.在映像服务中列出映像以验证与影像服务的连接：
```shell
[root@controller ~]# openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| c7a0e54f-d5d6-42a5-bda2-4090adc1c226 | cirros | active |
+--------------------------------------+--------+--------+
```
## 5.检查单元格和放置 API 工作顺利，以及其他必要的先决条件已到位：
```shell
[root@controller ~]# nova-status upgrade check
+--------------------------------------------------------------------+
| Upgrade Check Results                                              |
+--------------------------------------------------------------------+
| Check: Cells v2                                                    |
| Result: Success                                                    |
| Details: No host mappings or compute nodes were found. Remember to |
|   run command 'nova-manage cell_v2 discover_hosts' when new        |
|   compute hosts are deployed.                                      |
+--------------------------------------------------------------------+
| Check: Placement API                                               |
| Result: Success                                                    |
| Details: None                                                      |
+--------------------------------------------------------------------+
| Check: Ironic Flavor Migration                                     |
| Result: Success                                                    |
| Details: None                                                      |
+--------------------------------------------------------------------+
| Check: Cinder API                                                  |
| Result: Success                                                    |
| Details: None                                                      |
+--------------------------------------------------------------------+
```



 
