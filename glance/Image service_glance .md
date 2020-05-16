# glance
# 实验环境
- 在安装和配置映像服务之前，必须创建数据库、服务凭据和 API 终结点。
### 1.创建数据库
- 使用数据库访问客户端以用户身份连接到数据库服务器：root
```shell
[root@controller ~]# mysql -u root -pCom.123
```
- 创建数据库：glance
```shell
MariaDB [(none)]> CREATE DATABASE glance;
Query OK, 1 row affected (0.001 sec)
```
- 授予对数据库的适当访问权限：glance
```shell
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'Com.123';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'Com.123';
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> exit
Bye
```
### 2.源凭据以访问仅管理员 CLI 命令：admin
```shell
[root@controller ~]# . admin-openrc 
```
### 3.要创建服务凭据，请完成以下步骤：
- 创建用户：glance
- password:Com.123
```shell
[root@controller ~]# openstack user create --domain default --password-prompt glance
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 4d4a93dd0a6c4abbb754759ba7200eb2 |
| name                | glance                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```
- 将角色添加到用户和项目：admin glance service
```shell
[root@controller ~]# openstack role add --project service --user glance admin
```
- 创建服务实体：glance
```shell
[root@controller ~]# openstack service create --name glance   --description "OpenStack Image" image
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | 115e557bc8c14c9c92854f0379ca8a9e |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+
[root@controller ~]# 
```
### 4.创建影像服务 API 终结点：
```shell
[root@controller ~]# openstack endpoint create --region RegionOne image public http://controller:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | ca5da90616d548e18397b3dbecc8aadb |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 115e557bc8c14c9c92854f0379ca8a9e |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne image internal http://controller:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | de8a12eda1304a3bb0f1fc9dc319af56 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 115e557bc8c14c9c92854f0379ca8a9e |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne  image admin http://controller:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 80597c730c814890ac992895e98ceee4 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 115e557bc8c14c9c92854f0379ca8a9e |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+
```
##  一、安装和配置
### 1.安装软件包
```shell
[root@controller ~]# yum install openstack-glance  -y
```
### 2.编辑文件并完成以下操作：**/etc/glance/glance-api.conf**
```shell
# 配置数据库访问
[database]
connection = mysql+pymysql://glance:Com.123456@controller/glance
# 配置标识服务访问
[keystone_authtoken]
www_authenticate_uri  = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = Com.123456
[paste_deploy]
flavor = keystone
# 配置本地文件系统存储和映像文件的位置：[glance_store]
[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```
### 3.填充影像服务数据库：
```shell
[root@controller ~]# su -s /bin/sh -c "glance-manage db_sync" glance
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
/usr/lib/python2.7/site-packages/pymysql/cursors.py:170: Warning: (1280, u"Name 'alembic_version_pkc' ignored for PRIMARY key.")
  result = self._query(query)
INFO  [alembic.runtime.migration] Running upgrade  -> liberty, liberty initial
INFO  [alembic.runtime.migration] Running upgrade liberty -> mitaka01, add index on created_at and updated_at columns of 'images' table
INFO  [alembic.runtime.migration] Running upgrade mitaka01 -> mitaka02, update metadef os_nova_server
INFO  [alembic.runtime.migration] Running upgrade mitaka02 -> ocata_expand01, add visibility to images
INFO  [alembic.runtime.migration] Running upgrade ocata_expand01 -> pike_expand01, empty expand for symmetry with pike_contract01
INFO  [alembic.runtime.migration] Running upgrade pike_expand01 -> queens_expand01
INFO  [alembic.runtime.migration] Running upgrade queens_expand01 -> rocky_expand01, add os_hidden column to images table
INFO  [alembic.runtime.migration] Running upgrade rocky_expand01 -> rocky_expand02, add os_hash_algo and os_hash_value columns to images table
INFO  [alembic.runtime.migration] Running upgrade rocky_expand02 -> train_expand01, empty expand for symmetry with train_contract01
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
Upgraded database to: train_expand01, current revision(s): train_expand01
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
Database migration is up to date. No migration needed.
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade mitaka02 -> ocata_contract01, remove is_public from images
INFO  [alembic.runtime.migration] Running upgrade ocata_contract01 -> pike_contract01, drop glare artifacts tables
INFO  [alembic.runtime.migration] Running upgrade pike_contract01 -> queens_contract01
INFO  [alembic.runtime.migration] Running upgrade queens_contract01 -> rocky_contract01
INFO  [alembic.runtime.migration] Running upgrade rocky_contract01 -> rocky_contract02
INFO  [alembic.runtime.migration] Running upgrade rocky_contract02 -> train_contract01
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
Upgraded database to: train_contract01, current revision(s): train_contract01
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
Database is synced successfully.
```
### 4.启动服务，完成配置
```shell
[root@controller ~]# systemctl enable openstack-glance-api.service
[root@controller ~]# systemctl start  openstack-glance-api.service
```
##  二、验证操作
- ## token  https://www.sdnlab.com/17872.html 
![token.png](https://ae02.alicdn.com/kf/Hfbd9ad5d7ae0428d923fbae6ba993c03J.png)
### 1.源凭据以访问仅管理员 CLI 命令：admin
```shell
[root@controller ~]# . admin-openrc 
```
### 2.下载源映像：
- -c  采用断点续传
```shell
[root@controller ~]# wget  -c  http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
--2020-05-14 13:03:28--  https://github.com/cirros-dev/cirros/releases/tag/0.4.0/cirros-0.4.0-x86_64-disk.img
正在解析主机 github.com (github.com)... 13.229.188.59
正在连接 github.com (github.com)|13.229.188.59|:443... 已连接。
已发出 HTTP 请求，正在等待回应... 200 OK
长度：未指定 [text/html]
正在保存至: “cirros-0.4.0-x86_64-disk.img”

    [                            <=>                                                                                       ] 124,507     4.93KB/s 用时 20s    
2020-05-14 13:03:52 (6.00 KB/s) - “cirros-0.4.0-x86_64-disk.img” 已保存 [124507]
[root@controller ~]# ll cirros-0.4.0-x86_64-disk.img  -ld
-rw-r--r-- 1 root root 12716032 5月  15 10:47 cirros-0.4.0-x86_64-disk.img
```
### 3.使用QCOW2磁盘格式、裸容器格式和公共可见性将映像上载到映像服务，以便所有项目都可以访问它：
```shell
[root@controller ~]# glance image-create --name "cirros"   --file cirros-0.4.0-x86_64-disk.img   --disk-format qcow2 --container-format bare   --visibility public
+------------------+----------------------------------------------------------------------------------+
| Property         | Value                                                                            |
+------------------+----------------------------------------------------------------------------------+
| checksum         | 443b7623e27ecf03dc9e01ee93f67afe                                                 |
| container_format | bare                                                                             |
| created_at       | 2020-05-15T03:29:45Z                                                             |
| disk_format      | qcow2                                                                            |
| id               | c7a0e54f-d5d6-42a5-bda2-4090adc1c226                                             |
| min_disk         | 0                                                                                |
| min_ram          | 0                                                                                |
| name             | cirros                                                                           |
| os_hash_algo     | sha512                                                                           |
| os_hash_value    | 6513f21e44aa3da349f248188a44bc304a3653a04122d8fb4535423c8e1d14cd6a153f735bb0982e |
|                  | 2161b5b5186106570c17a9e58b64dd39390617cd5a350f78                                 |
| os_hidden        | False                                                                            |
| owner            | cc6e94f0881e45a28b21b14b4f310810                                                 |
| protected        | False                                                                            |
| size             | 12716032                                                                         |
| status           | active                                                                           |
| tags             | []                                                                               |
| updated_at       | 2020-05-15T03:29:46Z                                                             |
| virtual_size     | Not available                                                                    |
| visibility       | public                                                                           |
+------------------+----------------------------------------------------------------------------------+
```
### 4.确认图像的上传并验证属性：
```shell
[root@controller ~]# glance image-list
+--------------------------------------+--------+
| ID                                   | Name   |
+--------------------------------------+--------+
| c7a0e54f-d5d6-42a5-bda2-4090adc1c226 | cirros |
+--------------------------------------+--------+
```