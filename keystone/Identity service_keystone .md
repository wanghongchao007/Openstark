## keystone 

**在控制器节点上安装和配置代号为Keystone的OpenStack身份服务。为了实现可伸缩性，此配置部署了Fernet令牌和Apache HTTP服务器来处理请求**
### 1. 安装配置
  - 前提条件(在安装和配置身份服务之前，必须创建数据库）
```nginx
# 1.使用数据库访问客户端以root用户身份连接到数据库服务器：
[root@controller ~]# mysql -u root -pCom.123
# 2.创建keystone数据库：
MariaDB [(none)]> CREATE DATABASE keystone;
Query OK, 1 row affected (0.20 sec)
# 3.授予对keystone数据库的适当访问权限：
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost'  IDENTIFIED BY 'Com.123';
Query OK, 0 rows affected (0.01 sec)
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'Com.123';
Query OK, 0 rows affected (0.00 sec)
MariaDB [(none)]> exit
Bye
# 1.安装依赖包
[root@controller ~]# yum install openstack-keystone httpd mod_wsgi -y
# 2.编辑/etc/keystone/keystone.conf文件并完成以下操作：
在该[database]部分中，配置数据库访问：
[root@controller ~]# vim  /etc/keystone/keystone.conf
## 723行
[database]
connection = mysql+pymysql://keystone:Com.123456@controller/keystone
在该[token]部分中，配置Fernet令牌提供者：
## 2805行
[token]
provider = fernet
# 3.填充身份服务数据库：
[root@controller ~]# su -s /bin/sh -c "keystone-manage db_sync" keystone
# 4.初始化Fernet密钥存储库：
[root@controller ~]# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
[root@controller ~]# keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
# 5.引导身份服务：
[root@controller ~]# keystone-manage bootstrap --bootstrap-password Com.123 \
> --bootstrap-admin-url http://controller:5000/v3/ \
> --bootstrap-internal-url http://controller:5000/v3/ \
> --bootstrap-public-url http://controller:5000/v3/ \
> --bootstrap-region-id RegionOne
```
### 2. 配置Apache HTTP服务器
```nginx
# 1.编辑/etc/httpd/conf/httpd.conf文件并配置 ServerName选项以引用控制器节点：如果该ServerName条目尚不存在，则需要添加。
[root@controller ~]# vim  /etc/httpd/conf/httpd.conf
ServerName controller
# 2.创建到/usr/share/keystone/wsgi-keystone.conf文件的链接：
[root@controller ~]# ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
# 3.启动Apache HTTP服务，并将其配置为在系统启动时启动：
[root@controller ~]# systemctl enable httpd.service
[root@controller ~]# systemctl start httpd.service
# 4.通过设置适当的环境变量来配置管理帐户：
[root@controller ~]# export OS_USERNAME=admin
[root@controller ~]# export OS_PASSWORD=ADMIN_PASS
[root@controller ~]# export OS_PROJECT_NAME=admin
[root@controller ~]# export OS_USER_DOMAIN_NAME=Default
[root@controller ~]# export OS_PROJECT_DOMAIN_NAME=Default
[root@controller ~]# export OS_AUTH_URL=http://controller:5000/v3
[root@controller ~]# export OS_IDENTITY_API_VERSION=3

export OS_USERNAME=admin
export OS_PASSWORD=Com.123
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
```
### 3. 创建域，项目，用户和角色
```nginx
# 1.创建example域
[root@controller ~]# openstack domain create --description "An Example Domain" example
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | An Example Domain                |
| enabled     | True                             |
| id          | 54257ed9412745f29b7ba6dc63374749 |
| name        | example                          |
| options     | {}                               |
| tags        | []                               |
+-------------+----------------------------------+
# 2.添加到环境的每个服务的唯一用户。创建项目：service
[root@controller ~]# openstack project create --domain default  --description "Service Project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | f56cae2f1f2844969c70b853d5e5fe95 |
| is_domain   | False                            |
| name        | service                          |
| options     | {}                               |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+
# 3.创建项目和用户
# 3.1.创建项目：myproject
[root@controller ~]# openstack project create --domain default --description "Demo Project" myproject
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | default                          |
| enabled     | True                             |
| id          | e9d3bca621114162b553953b443e7655 |
| is_domain   | False                            |
| name        | myproject                        |
| options     | {}                               |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+
# 3.2.创建用户：myuser
[root@controller ~]# openstack user create --domain default --password-prompt myuser
User Password:  Com.123
Repeat User Password:  Com.123
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 7851b96c434e45e8aac0c067a2ea8ce6 |
| name                | myuser                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
# 3.3.创建角色：myrole
[root@controller ~]# openstack role create myrole
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | None                             |
| domain_id   | None                             |
| id          | a1055e74de7149dfa3cfa2da252a863d |
| name        | myrole                           |
| options     | {}                               |
+-------------+----------------------------------+
# 3.4.将角色添加到项目和用户：myrole  myproject myuser
[root@controller ~]# openstack role add --project myproject --user myuser myrole
```
### 4.  验证操作
```nginx
# 1.取消设置临时OS_AUTH_URL和OS_PASSWORD环境变量：
[root@controller ~]# unset OS_AUTH_URL OS_PASSWORD
# 2.作为用户，请求身份验证令牌：admin
[root@controller ~]# openstack --os-auth-url http://controller:5000/v3 \
>  --os-project-domain-name Default --os-user-domain-name Default \
> --os-project-name admin --os-username admin token issue
Password: Com.123
Password: Com.123
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2020-05-15T04:11:34+0000                                                                                                                                                                |
| id         | gAAAAABevghmSSyQv4KoqFyjGkHq5ChehXoL5Uf78ZaT1IdbFRkUZPZJrRCU5uIjSrsFoSFWT9D_E0nTrvCzIkwQZB4_fW35O9h_nzs0pdLDsvsz2h6OnMK00mdXibY8Lgx57GxVNywde2dE_J4bN5WtnTcU_uIxoj18lXdXDCpsSWWHkptMdz8 |
| project_id | cc6e94f0881e45a28b21b14b4f310810                                                                                                                                                        |
| user_id    | 38493bce7c774b01b5a7b488d1aabfac                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+


# 3.作为上一节中创建的myuser用户，请求身份验证令牌：
[root@controller ~]# openstack --os-auth-url http://controller:5000/v3  \
> --os-project-domain-name Default --os-user-domain-name Default \
> --os-project-name myproject --os-username myuser token issue
Password: Com.123
Password: Com.123
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2020-05-15T04:13:56+0000                                                                                                                                                                |
| id         | gAAAAABevgj0JwfXGaIVVTJSmdUQ1C56ZCC3Ulu7TK6JRlBanIR-MrX9N4OJ0JU_q6rBhrxQS0l-bHJFy5t7W-iUoH6EeWOdV7PDqYLnkzR05SuRWjGQqcLrgIvrmtXx0V4uSAs1GAD8m0HsSvMxskpIOjmsXOMrjg3c3ygvvP2YiSXeIlRIxSM |
| project_id | 139f19dd84484ce5b5f98c704e6216dc                                                                                                                                                        |
| user_id    | 8c467c37ac9641fba0d8c6aea82de4fe                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```
### 5.  建OpenStack客户端环境脚本
#### 5.1.创建脚本
- 为项目和用户创建客户端环境脚本
- 引用这些脚本来加载客户端操作的适当凭据admindemo
  - 创建和编辑文件并添加以下内容：admin-openrc
  - 创建和编辑文件并添加以下内容：demo-openrc
```nginx
[root@controller ~]# vim  admin-openrc
[root@controller ~]# cat  admin-openrc 
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=Com.123
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```
```nginx
[root@controller ~]# vim  demo-openrc
[root@controller ~]# cat  demo-openrc 
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=myproject
export OS_USERNAME=myuser
export OS_PASSWORD=Com.123
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```
#### 5.2. 使用脚本
- 1.加载文件以使用标识服务的位置以及项目和用户凭据填充环境变量：admin-openrcadmin 
```nginx
[root@controller ~]# . admin-openrc 
```
-  2.请求身份验证令牌：
```nginx
[root@controller ~]# openstack token issue
[root@controller ~]# openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2020-05-15T04:15:34+0000                                                                                                                                                                |
| id         | gAAAAABevglW4O6uRiR53Rz32CgEeqRlxxJXMitpFX5CMHhQFNUFDWVATNqs90_7d1oKbdwvjspECGykhM0dV6WkBryDTSXR03yWdTsGezp2bTt0a_Y_aWY1T_TLMQp2ZnPdBmB6hAYRF_bhIurQ6WbQbLOdHqnNzbOAM86hhqN59fHTxh2Soss |
| project_id | cc6e94f0881e45a28b21b14b4f310810                                                                                                                                                        |
| user_id    | 38493bce7c774b01b5a7b488d1aabfac                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```