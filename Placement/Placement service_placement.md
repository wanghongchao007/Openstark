#  安装放置服务
https://docs.openstack.org/placement/train/install/install-rdo.html 
#### 实验环境准备
- 在安装和配置放置服务之前，必须创建数据库、服务凭据和 API 终结点。
## 1.安装数据库
### 1.1.使用数据库访问客户端以用户身份连接到数据库服务器：root
```shell
[root@controller ~]# mysql -u root -pCom.123
```
### 1.2.创建数据库：placement
```shell
MariaDB [(none)]> CREATE DATABASE placement;
Query OK, 1 row affected (0.000 sec)
```
### 1.3.授予对数据库的适当访问权限：
```shell
MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'Com.123456';
Query OK, 0 rows affected (0.001 sec)
MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%'  IDENTIFIED BY 'Com.123456';
Query OK, 0 rows affected (0.000 sec)
```
### 2.配置用户和终结点
### 2.1.源凭据以访问仅管理员 CLI 命令：admin
```shell
[root@controller ~]# . admin-openrc 
```
### 2.2.使用您选择的创建安置服务用户：
- password =Com.123

```shell
[root@controller ~]# openstack user create --domain default --password-prompt placement
User Password: Com.123
Repeat User Password: Com.123
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | acf520b7220d467890ddb2c7ab767c83 |
| name                | placement                        |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```
### 2.3.使用管理员角色将展示用户添加到服务项目：
```shell
[root@controller ~]# openstack role add --project service --user placement admin
```
### 2.4.在服务目录中创建放置 API 条目：
```shell
[root@controller ~]# openstack service create --name placement --description "Placement API" placement
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Placement API                    |
| enabled     | True                             |
| id          | ad6ef6a90aa1447f8c126484658320eb |
| name        | placement                        |
| type        | placement                        |
+-------------+----------------------------------+
```
### 2.5.创建放置 API 服务终结点：
```shell
[root@controller ~]# openstack endpoint create --region RegionOne placement public http://controller:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | d1d535615b824431a058632ef3afaec3 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | ad6ef6a90aa1447f8c126484658320eb |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne placement internal http://controller:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 8ccc15f796f845edb9e1b7f2c6c11114 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | ad6ef6a90aa1447f8c126484658320eb |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne placement admin http://controller:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 27784ecde84d464ea24f49fa3b029de8 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | ad6ef6a90aa1447f8c126484658320eb |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+
```
## 3.安装和配置组件
### 3.1.安装软件包： 
```shell
[root@controller ~]# yum install openstack-placement-api  -y 
```
### 3.2.编辑文件并完成以下操作：**/etc/placement/placement.conf**
- 配置数据库访问：[placement_database]
- 配置标识服务访问：[api][keystone_authtoken]
```shell
[root@controller ~]# vim  /etc/placement/placement.conf
# 配置数据库访问
[placement_database]
connection = mysql+pymysql://placement:Com.123456@controller/placement
# 配置标识服务访问
[api]
auth_strategy = keystone
[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = Com.123



[root@controller ~]# grep -v '^$' /etc/placement/placement.conf | grep -v '^#'
[DEFAULT]
[api]
auth_strategy = keystone
[cors]
[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = Com.123
[oslo_policy]
[placement]
[placement_database]
connection = mysql+pymysql://placement:Com.123456@controller/placement
[profiler]
```
### 3.3.填充数据库：placement
```shell
[root@controller ~]# su -s /bin/sh -c "placement-manage db sync" placement
/usr/lib/python2.7/site-packages/pymysql/cursors.py:170: Warning: (1280, u"Name 'alembic_version_pkc' ignored for PRIMARY key.")
  result = self._query(query)
```
## 4.完成安装
- 重启httpd服务
- 开机自启服务
- 服务状态
```shell
[root@controller ~]# systemctl enable httpd.service 
[root@controller ~]# systemctl start  httpd.service 
[root@controller ~]# systemctl status  httpd.service 
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; vendor preset: disabled)
   Active: active (running) since 四 2020-05-14 14:47:53 CST; 16s ago
     Docs: man:httpd(8)
           man:apachectl(8)
 Main PID: 9315 (httpd)
   Status: "Total requests: 0; Current requests/sec: 0; Current traffic:   0 B/sec"
   CGroup: /system.slice/httpd.service
           ├─9315 /usr/sbin/httpd -DFOREGROUND
           ├─9316 /usr/sbin/httpd -DFOREGROUND
           ├─9317 /usr/sbin/httpd -DFOREGROUND
           ├─9318 /usr/sbin/httpd -DFOREGROUND
           ├─9319 (wsgi:keystone- -DFOREGROUND
           ├─9320 (wsgi:keystone- -DFOREGROUND
           ├─9321 (wsgi:keystone- -DFOREGROUND
           ├─9322 (wsgi:keystone- -DFOREGROUND
           ├─9323 (wsgi:keystone- -DFOREGROUND
           ├─9324 /usr/sbin/httpd -DFOREGROUND
           ├─9325 /usr/sbin/httpd -DFOREGROUND
           ├─9326 /usr/sbin/httpd -DFOREGROUND
           ├─9327 /usr/sbin/httpd -DFOREGROUND
           └─9328 /usr/sbin/httpd -DFOREGROUND

5月 14 14:47:53 controller systemd[1]: Starting The Apache HTTP Server...
5月 14 14:47:53 controller systemd[1]: Started The Apache HTTP Server.
```

## 5.验证安装
### 5.1.源凭据以访问仅管理员 CLI 命令：admin
```shell
[root@controller ~]# . admin-openrc 
```
### 5.2.执行状态检查以确保一切井然有序：
```shell
[root@controller ~]# placement-status upgrade check
+----------------------------------+
| Upgrade Check Results            |
+----------------------------------+
| Check: Missing Root Provider IDs |
| Result: Success                  |
| Details: None                    |
+----------------------------------+
| Check: Incomplete Consumers      |
| Result: Success                  |
| Details: None                    |
+----------------------------------+
```
### 5.3.针对放置 API 运行一些命令：
- 安装osc 放置插件：
- 列出可用的资源类和特征：
-  https://docs.openstack.org/osc-placement/latest/ 
```shell
[root@controller ~]# yum  install  epel-release -y
[root@controller ~]# yum install  python2-pip -y
[root@controller ~]# mkdir /root/.pip
[root@controller ~]# vim .pip/pip.conf
[root@controller ~]# cat .pip/pip.conf
[global] 
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host=pypi.tuna.tsinghua.edu.cn
[root@controller ~]# pip install osc-placement
Collecting osc-placement
  Downloading https://pypi.tuna.tsinghua.edu.cn/packages/0b/22/fa2bd745179a3ec06b719a392e844163f31ebb40f5f73cfe1b174a040daa/osc-placement-2.0.0.tar.gz (47kB)
    100% |████████████████████████████████| 51kB 290kB/s 
Requirement already satisfied (use --upgrade to upgrade): pbr>=2.0.0 in /usr/lib/python2.7/site-packages (from osc-placement)
Requirement already satisfied (use --upgrade to upgrade): six>=1.10.0 in /usr/lib/python2.7/site-packages (from osc-placement)
Requirement already satisfied (use --upgrade to upgrade): keystoneauth1>=3.3.0 in /usr/lib/python2.7/site-packages (from osc-placement)
Requirement already satisfied (use --upgrade to upgrade): simplejson>=3.16.0 in /usr/lib64/python2.7/site-packages (from osc-placement)
Requirement already satisfied (use --upgrade to upgrade): osc-lib>=1.2.0 in /usr/lib/python2.7/site-packages (from osc-placement)
Requirement already satisfied (use --upgrade to upgrade): oslo.utils>=3.37.0 in /usr/lib/python2.7/site-packages (from osc-placement)
Installing collected packages: osc-placement
  Running setup.py install for osc-placement ... done
Successfully installed osc-placement-2.0.0



[root@controller ~]# openstack --os-placement-api-version 1.4  resource class list --sort-column name
+----------------------------+
| name                       |
+----------------------------+
| DISK_GB                    |
| FPGA                       |
| IPV4_ADDRESS               |
| MEMORY_MB                  |
| MEM_ENCRYPTION_CONTEXT     |
| NET_BW_EGR_KILOBIT_PER_SEC |
| NET_BW_IGR_KILOBIT_PER_SEC |
| NUMA_CORE                  |
| NUMA_MEMORY_MB             |
| NUMA_SOCKET                |
| NUMA_THREAD                |
| PCI_DEVICE                 |
| PCPU                       |
| PGPU                       |
| SRIOV_NET_VF               |
| VCPU                       |
| VGPU                       |
| VGPU_DISPLAY_HEAD          |
+----------------------------+

[root@controller ~]# openstack --os-placement-api-version 1.6 trait list --sort-column name
+---------------------------------------+
| name                                  |
+---------------------------------------+
| COMPUTE_DEVICE_TAGGING                |
| COMPUTE_GRAPHICS_MODEL_CIRRUS         |
| COMPUTE_GRAPHICS_MODEL_GOP            |
| COMPUTE_GRAPHICS_MODEL_NONE           |
| COMPUTE_GRAPHICS_MODEL_QXL            |
| COMPUTE_GRAPHICS_MODEL_VGA            |
| COMPUTE_GRAPHICS_MODEL_VIRTIO         |
| COMPUTE_GRAPHICS_MODEL_VMVGA          |
| COMPUTE_GRAPHICS_MODEL_XEN            |
| COMPUTE_IMAGE_TYPE_AKI                |
| COMPUTE_IMAGE_TYPE_AMI                |
| COMPUTE_IMAGE_TYPE_ARI                |
| COMPUTE_IMAGE_TYPE_ISO                |
| COMPUTE_IMAGE_TYPE_QCOW2              |
| COMPUTE_IMAGE_TYPE_RAW                |
| COMPUTE_IMAGE_TYPE_VDI                |
| COMPUTE_IMAGE_TYPE_VHD                |
| COMPUTE_IMAGE_TYPE_VHDX               |
| COMPUTE_IMAGE_TYPE_VMDK               |
| COMPUTE_MIGRATE_AUTO_CONVERGE         |
| COMPUTE_MIGRATE_POST_COPY             |
| COMPUTE_NET_ATTACH_INTERFACE          |
| COMPUTE_NET_ATTACH_INTERFACE_WITH_TAG |
| COMPUTE_NET_VIF_MODEL_E1000           |
| COMPUTE_NET_VIF_MODEL_E1000E          |
| COMPUTE_NET_VIF_MODEL_LAN9118         |
| COMPUTE_NET_VIF_MODEL_NE2K_PCI        |
| COMPUTE_NET_VIF_MODEL_NETFRONT        |
| COMPUTE_NET_VIF_MODEL_PCNET           |
| COMPUTE_NET_VIF_MODEL_RTL8139         |
| COMPUTE_NET_VIF_MODEL_SPAPR_VLAN      |
| COMPUTE_NET_VIF_MODEL_SRIOV           |
| COMPUTE_NET_VIF_MODEL_VIRTIO          |
| COMPUTE_NET_VIF_MODEL_VMXNET          |
| COMPUTE_NET_VIF_MODEL_VMXNET3         |
| COMPUTE_SECURITY_TPM_1_2              |
| COMPUTE_SECURITY_TPM_2_0              |
| COMPUTE_STATUS_DISABLED               |
| COMPUTE_STORAGE_BUS_FDC               |
| COMPUTE_STORAGE_BUS_IDE               |
| COMPUTE_STORAGE_BUS_LXC               |
| COMPUTE_STORAGE_BUS_SATA              |
| COMPUTE_STORAGE_BUS_SCSI              |
| COMPUTE_STORAGE_BUS_UML               |
| COMPUTE_STORAGE_BUS_USB               |
| COMPUTE_STORAGE_BUS_VIRTIO            |
| COMPUTE_STORAGE_BUS_XEN               |
| COMPUTE_TRUSTED_CERTS                 |
| COMPUTE_VOLUME_ATTACH                 |
| COMPUTE_VOLUME_ATTACH_WITH_TAG        |
| COMPUTE_VOLUME_EXTEND                 |
| COMPUTE_VOLUME_MULTI_ATTACH           |
| HW_CPU_AARCH64_AES                    |
| HW_CPU_AARCH64_ASIMD                  |
| HW_CPU_AARCH64_ASIMDDP                |
| HW_CPU_AARCH64_ASIMDHP                |
| HW_CPU_AARCH64_ASIMDRDM               |
| HW_CPU_AARCH64_ATOMICS                |
| HW_CPU_AARCH64_CPUID                  |
| HW_CPU_AARCH64_CRC32                  |
| HW_CPU_AARCH64_DCPOP                  |
| HW_CPU_AARCH64_EVTSTRM                |
| HW_CPU_AARCH64_FCMA                   |
| HW_CPU_AARCH64_FP                     |
| HW_CPU_AARCH64_FPHP                   |
| HW_CPU_AARCH64_JSCVT                  |
| HW_CPU_AARCH64_LRCPC                  |
| HW_CPU_AARCH64_PMULL                  |
| HW_CPU_AARCH64_SHA1                   |
| HW_CPU_AARCH64_SHA2                   |
| HW_CPU_AARCH64_SHA3                   |
| HW_CPU_AARCH64_SHA512                 |
| HW_CPU_AARCH64_SM3                    |
| HW_CPU_AARCH64_SM4                    |
| HW_CPU_AARCH64_SVE                    |
| HW_CPU_AMD_SEV                        |
| HW_CPU_HYPERTHREADING                 |
| HW_CPU_X86_3DNOW                      |
| HW_CPU_X86_ABM                        |
| HW_CPU_X86_AESNI                      |
| HW_CPU_X86_AMD_IBPB                   |
| HW_CPU_X86_AMD_NO_SSB                 |
| HW_CPU_X86_AMD_SEV                    |
| HW_CPU_X86_AMD_SSBD                   |
| HW_CPU_X86_AMD_SVM                    |
| HW_CPU_X86_AMD_VIRT_SSBD              |
| HW_CPU_X86_ASF                        |
| HW_CPU_X86_AVX                        |
| HW_CPU_X86_AVX2                       |
| HW_CPU_X86_AVX512BW                   |
| HW_CPU_X86_AVX512CD                   |
| HW_CPU_X86_AVX512DQ                   |
| HW_CPU_X86_AVX512ER                   |
| HW_CPU_X86_AVX512F                    |
| HW_CPU_X86_AVX512PF                   |
| HW_CPU_X86_AVX512VL                   |
| HW_CPU_X86_AVX512VNNI                 |
| HW_CPU_X86_BMI                        |
| HW_CPU_X86_BMI2                       |
| HW_CPU_X86_CLMUL                      |
| HW_CPU_X86_F16C                       |
| HW_CPU_X86_FMA3                       |
| HW_CPU_X86_FMA4                       |
| HW_CPU_X86_INTEL_MD_CLEAR             |
| HW_CPU_X86_INTEL_PCID                 |
| HW_CPU_X86_INTEL_SPEC_CTRL            |
| HW_CPU_X86_INTEL_SSBD                 |
| HW_CPU_X86_INTEL_VMX                  |
| HW_CPU_X86_MMX                        |
| HW_CPU_X86_MPX                        |
| HW_CPU_X86_PDPE1GB                    |
| HW_CPU_X86_SGX                        |
| HW_CPU_X86_SHA                        |
| HW_CPU_X86_SSE                        |
| HW_CPU_X86_SSE2                       |
| HW_CPU_X86_SSE3                       |
| HW_CPU_X86_SSE41                      |
| HW_CPU_X86_SSE42                      |
| HW_CPU_X86_SSE4A                      |
| HW_CPU_X86_SSSE3                      |
| HW_CPU_X86_STIBP                      |
| HW_CPU_X86_SVM                        |
| HW_CPU_X86_TBM                        |
| HW_CPU_X86_TSX                        |
| HW_CPU_X86_VMX                        |
| HW_CPU_X86_XOP                        |
| HW_GPU_API_DIRECT2D                   |
| HW_GPU_API_DIRECT3D_V10_0             |
| HW_GPU_API_DIRECT3D_V10_1             |
| HW_GPU_API_DIRECT3D_V11_0             |
| HW_GPU_API_DIRECT3D_V11_1             |
| HW_GPU_API_DIRECT3D_V11_2             |
| HW_GPU_API_DIRECT3D_V11_3             |
| HW_GPU_API_DIRECT3D_V12_0             |
| HW_GPU_API_DIRECT3D_V6_0              |
| HW_GPU_API_DIRECT3D_V7_0              |
| HW_GPU_API_DIRECT3D_V8_0              |
| HW_GPU_API_DIRECT3D_V8_1              |
| HW_GPU_API_DIRECT3D_V9_0              |
| HW_GPU_API_DIRECT3D_V9_0B             |
| HW_GPU_API_DIRECT3D_V9_0C             |
| HW_GPU_API_DIRECT3D_V9_0L             |
| HW_GPU_API_DIRECTX_V10                |
| HW_GPU_API_DIRECTX_V11                |
| HW_GPU_API_DIRECTX_V12                |
| HW_GPU_API_DXVA                       |
| HW_GPU_API_OPENCL_V1_0                |
| HW_GPU_API_OPENCL_V1_1                |
| HW_GPU_API_OPENCL_V1_2                |
| HW_GPU_API_OPENCL_V2_0                |
| HW_GPU_API_OPENCL_V2_1                |
| HW_GPU_API_OPENCL_V2_2                |
| HW_GPU_API_OPENGL_V1_1                |
| HW_GPU_API_OPENGL_V1_2                |
| HW_GPU_API_OPENGL_V1_3                |
| HW_GPU_API_OPENGL_V1_4                |
| HW_GPU_API_OPENGL_V1_5                |
| HW_GPU_API_OPENGL_V2_0                |
| HW_GPU_API_OPENGL_V2_1                |
| HW_GPU_API_OPENGL_V3_0                |
| HW_GPU_API_OPENGL_V3_1                |
| HW_GPU_API_OPENGL_V3_2                |
| HW_GPU_API_OPENGL_V3_3                |
| HW_GPU_API_OPENGL_V4_0                |
| HW_GPU_API_OPENGL_V4_1                |
| HW_GPU_API_OPENGL_V4_2                |
| HW_GPU_API_OPENGL_V4_3                |
| HW_GPU_API_OPENGL_V4_4                |
| HW_GPU_API_OPENGL_V4_5                |
| HW_GPU_API_VULKAN                     |
| HW_GPU_CUDA_COMPUTE_CAPABILITY_V1_0   |
| HW_GPU_CUDA_COMPUTE_CAPABILITY_V1_1   |
| HW_GPU_CUDA_COMPUTE_CAPABILITY_V1_2   |
| HW_GPU_CUDA_COMPUTE_CAPABILITY_V1_3   |
| HW_GPU_CUDA_COMPUTE_CAPABILITY_V2_0   |
| HW_GPU_CUDA_COMPUTE_CAPABILITY_V2_1   |
| HW_GPU_CUDA_COMPUTE_CAPABILITY_V3_0   |
| HW_GPU_CUDA_COMPUTE_CAPABILITY_V3_2   |
| HW_GPU_CUDA_COMPUTE_CAPABILITY_V3_5   |
| HW_GPU_CUDA_COMPUTE_CAPABILITY_V3_7   |
| HW_GPU_CUDA_COMPUTE_CAPABILITY_V5_0   |
| HW_GPU_CUDA_COMPUTE_CAPABILITY_V5_2   |
| HW_GPU_CUDA_COMPUTE_CAPABILITY_V5_3   |
| HW_GPU_CUDA_COMPUTE_CAPABILITY_V6_0   |
| HW_GPU_CUDA_COMPUTE_CAPABILITY_V6_1   |
| HW_GPU_CUDA_COMPUTE_CAPABILITY_V6_2   |
| HW_GPU_CUDA_COMPUTE_CAPABILITY_V7_0   |
| HW_GPU_CUDA_COMPUTE_CAPABILITY_V7_1   |
| HW_GPU_CUDA_COMPUTE_CAPABILITY_V7_2   |
| HW_GPU_CUDA_SDK_V10_0                 |
| HW_GPU_CUDA_SDK_V6_5                  |
| HW_GPU_CUDA_SDK_V7_5                  |
| HW_GPU_CUDA_SDK_V8_0                  |
| HW_GPU_CUDA_SDK_V9_0                  |
| HW_GPU_CUDA_SDK_V9_1                  |
| HW_GPU_CUDA_SDK_V9_2                  |
| HW_GPU_MAX_DISPLAY_HEADS_1            |
| HW_GPU_MAX_DISPLAY_HEADS_2            |
| HW_GPU_MAX_DISPLAY_HEADS_4            |
| HW_GPU_MAX_DISPLAY_HEADS_6            |
| HW_GPU_MAX_DISPLAY_HEADS_8            |
| HW_GPU_RESOLUTION_W1024H600           |
| HW_GPU_RESOLUTION_W1024H768           |
| HW_GPU_RESOLUTION_W1152H864           |
| HW_GPU_RESOLUTION_W1280H1024          |
| HW_GPU_RESOLUTION_W1280H720           |
| HW_GPU_RESOLUTION_W1280H768           |
| HW_GPU_RESOLUTION_W1280H800           |
| HW_GPU_RESOLUTION_W1360H768           |
| HW_GPU_RESOLUTION_W1366H768           |
| HW_GPU_RESOLUTION_W1440H900           |
| HW_GPU_RESOLUTION_W1600H1200          |
| HW_GPU_RESOLUTION_W1600H900           |
| HW_GPU_RESOLUTION_W1680H1050          |
| HW_GPU_RESOLUTION_W1920H1080          |
| HW_GPU_RESOLUTION_W1920H1200          |
| HW_GPU_RESOLUTION_W2560H1440          |
| HW_GPU_RESOLUTION_W2560H1600          |
| HW_GPU_RESOLUTION_W320H240            |
| HW_GPU_RESOLUTION_W3840H2160          |
| HW_GPU_RESOLUTION_W640H480            |
| HW_GPU_RESOLUTION_W7680H4320          |
| HW_GPU_RESOLUTION_W800H600            |
| HW_NIC_ACCEL_DEFLATE                  |
| HW_NIC_ACCEL_DIFFIEH                  |
| HW_NIC_ACCEL_ECC                      |
| HW_NIC_ACCEL_IPSEC                    |
| HW_NIC_ACCEL_LZS                      |
| HW_NIC_ACCEL_RSA                      |
| HW_NIC_ACCEL_SSL                      |
| HW_NIC_ACCEL_TLS                      |
| HW_NIC_DCB_ETS                        |
| HW_NIC_DCB_PFC                        |
| HW_NIC_DCB_QCN                        |
| HW_NIC_MULTIQUEUE                     |
| HW_NIC_OFFLOAD_FDF                    |
| HW_NIC_OFFLOAD_GENEVE                 |
| HW_NIC_OFFLOAD_GRE                    |
| HW_NIC_OFFLOAD_GRO                    |
| HW_NIC_OFFLOAD_GSO                    |
| HW_NIC_OFFLOAD_L2CRC                  |
| HW_NIC_OFFLOAD_LRO                    |
| HW_NIC_OFFLOAD_LSO                    |
| HW_NIC_OFFLOAD_QINQ                   |
| HW_NIC_OFFLOAD_RDMA                   |
| HW_NIC_OFFLOAD_RX                     |
| HW_NIC_OFFLOAD_RXHASH                 |
| HW_NIC_OFFLOAD_RXVLAN                 |
| HW_NIC_OFFLOAD_SCS                    |
| HW_NIC_OFFLOAD_SG                     |
| HW_NIC_OFFLOAD_SWITCHDEV              |
| HW_NIC_OFFLOAD_TCS                    |
| HW_NIC_OFFLOAD_TSO                    |
| HW_NIC_OFFLOAD_TX                     |
| HW_NIC_OFFLOAD_TXUDP                  |
| HW_NIC_OFFLOAD_TXVLAN                 |
| HW_NIC_OFFLOAD_UCS                    |
| HW_NIC_OFFLOAD_UFO                    |
| HW_NIC_OFFLOAD_VXLAN                  |
| HW_NIC_PROGRAMMABLE_PIPELINE          |
| HW_NIC_SRIOV                          |
| HW_NIC_SRIOV_MULTIQUEUE               |
| HW_NIC_SRIOV_QOS_RX                   |
| HW_NIC_SRIOV_QOS_TX                   |
| HW_NIC_SRIOV_TRUSTED                  |
| HW_NIC_VMDQ                           |
| HW_NUMA_ROOT                          |
| MISC_SHARES_VIA_AGGREGATE             |
| STORAGE_DISK_HDD                      |
| STORAGE_DISK_SSD                      |
+---------------------------------------+
```
## 6.报错解决 Expecting value: line 1 column 1 (char 0)

### 6.1.错误分析
#### 这是因为我们在apache没有授权

```
[root@controller ~]# openstack --os-placement-api-version 1.4  resource class list --sort-column name
Expecting value: line 1 column 1 (char 0)

[root@controller ~]# vim /etc/httpd/conf.d/00-placement-api.conf
Listen 8778

<VirtualHost *:8778>
  WSGIProcessGroup placement-api
  WSGIApplicationGroup %{GLOBAL}
  WSGIPassAuthorization On
  WSGIDaemonProcess placement-api processes=3 threads=1 user=placement group=placement
  WSGIScriptAlias / /usr/bin/placement-api
  <IfVersion >= 2.4>
    ErrorLogFormat "%M"
  </IfVersion>
  ErrorLog /var/log/placement/placement-api.log
  #SSLEngine On
  #SSLCertificateFile ...
  #SSLCertificateKeyFile ...
  <Directory /usr/bin>
     <IfVersion >= 2.4>
       Require all granted
     </IfVersion>
     <IfVersion < 2.4>
       Order allow,deny
       Allow from all
     </IfVersion>
  </Directory>
</VirtualHost>
Alias /placement-api /usr/bin/placement-api
<Location /placement-api>
  SetHandler wsgi-script
  Options +ExecCGI
  WSGIProcessGroup placement-api
  WSGIApplicationGroup %{GLOBAL}
  WSGIPassAuthorization On
</Location>
```

```shell
[root@controller ~]# systemctl   restart  httpd.service 
[root@controller ~]# curl http://10.0.0.20:8778
{"versions": [{"status": "CURRENT", "min_version": "1.0", "max_version": "1.36", "id": "v1.0", "links": [{"href": "", "rel": "self"}]}]}
[root@controller ~]# openstack --os-placement-api-version 1.4  resource class list --sort-column name
+----------------------------+
| name                       |
+----------------------------+
| DISK_GB                    |
| FPGA                       |
| IPV4_ADDRESS               |
| MEMORY_MB                  |
| MEM_ENCRYPTION_CONTEXT     |
| NET_BW_EGR_KILOBIT_PER_SEC |
| NET_BW_IGR_KILOBIT_PER_SEC |
| NUMA_CORE                  |
| NUMA_MEMORY_MB             |
| NUMA_SOCKET                |
| NUMA_THREAD                |
| PCI_DEVICE                 |
| PCPU                       |
| PGPU                       |
| SRIOV_NET_VF               |
| VCPU                       |
| VGPU                       |
| VGPU_DISPLAY_HEAD          |
+----------------------------+
```

