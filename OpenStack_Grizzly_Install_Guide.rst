==========================================================
  OpenStack Grizzly 多节点安装
==========================================================

:Version: 2.0
:Source: https://github.com/mseknibilel/OpenStack-Grizzly-Install-Guide
:Keywords: Multi node, Grizzly, Quantum, Nova, Keystone, Glance, Horizon, Cinder, Swift, OpenVSwitch, KVM, Ubuntu Server 12.04 (64 bits).

作者
==========
`Fungo Wang <http://fungo.me/>`_ <`bo@fungo.me <bo@fungo.me>`_>

目录
=================

::

  0. 介绍
  1. 安装环境说明
  2. MQ节点
  3. 控制节点
  4. 网络节点
  5. 计算节点
  6. 块存储节点
  7. 对象存储节点
  8. 创建虚拟机
  9. 联系
  10. 致谢
  
0. 介绍
==============

本文档旨在帮助你轻松创建自己的OpenStack云平台。

状态: Stable

1. 安装环境说明
================

:节点角色: 网卡配置(NIC)
:MQ节点: eth0 (10.10.10.50), eth1 (172.16.9.150)
:控制节点: eth0 (10.10.10.51), eth1 (172.16.9.151)
:网络节点: eth0 (10.10.10.52), eth1 (10.20.20.52), eth2 (172.16.9.152)
:计算节点: eth0 (10.10.10.53), eth1 (10.20.20.53), eth2 (172.16.9.152)
:块存储节点: eth0 (10.10.10.54), eth1 (172.16.9.154)
:对象存储节点: eth0 (10.10.10.55), eth1 (172.16.9.155)



**Note 1:** 可以用 dpkg -s <packagename> 命令来确认你安装的是否是Grizzly版的OpenStack (version : 2013.1)

**Note 2:** 下面是网络架构图，

.. image:: http://i.imgur.com/Frsughe.jpg


2. MQ节点
===============


2.1. 操作系统
-----------------

* 安装ubuntu 12.04.2 Server版本，最小化安装，只需要安装SSH server就可以。安装完成之后切换进入root用户，本文档之后的所有命令都在root下执行::

   sudo su
    
* 升级系统::

   apt-get update -y && apt-get upgrade -y && apt-get dist-upgrade -y
   
2.2. hostname 设置(根据自己实际情况调整)
----------------------------------------

* 相应文件配置如下::

    # cat /etc/hostname 
    grizzly-mq
  
    # cat /etc/hosts
    127.0.0.1       localhost
    10.10.10.50     grizzly-mq.hengtiansoft.com     grizzly-mq

    # hostname
    grizzly-mq

    # hostname -f
    grizzly-mq.hengtiansoft.com

   
2.3. 网络设置
--------------

* 保证只有一个网卡能访问Internet::

    #For Downloading Deb Packages from Internet repo
    auto eth1
    iface eth1 inet static
            address 172.16.9.150
            netmask 255.255.255.0
            network 172.16.9.0
            broadcast 172.16.9.255
            gateway 172.16.9.254
            # dns-* options are implemented by the resolvconf package, if installed
            dns-nameservers 172.16.5.1
            dns-search hengtiansoft.com

    #Not internet connected(used for OpenStack management)
    auto eth0
        iface eth0 inet static
        address 10.10.10.50
        netmask 255.255.255.0

* 重启网络服务::

   /etc/init.d/networking restart

2.4. rabbitmq
--------------

* 安装 rabbitmq::

   apt-get install -y rabbitmq-server

* 验证 rabbitmq正常启动 ::

   service rabbitmq-server status

* Troubleshooting rabbitmq::

   如果 rabbitmq-server 不能正常启动，请检查 /etc/hosts 文件，查看里面的 IP 是否是本机IP，确保
   ping `hostname`
   有正常返回。


3. 控制节点
===============

3.1. 操作系统
-----------------

* 安装ubuntu 12.04.2 Server版本，最小化安装，只需要安装SSH server就可以。安装完成之后切换进入root用户，本文档之后的所有命令都在root下执行::

   sudo su
    
* 添加Grizzly仓库 [只适用于 Ubuntu 12.04]::

   apt-get update
   apt-get install ubuntu-cloud-keyring

   cat <<EOF >>/etc/apt/sources.list
   deb  http://ubuntu-cloud.archive.canonical.com/ubuntu precise-proposed/grizzly main
   deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/grizzly main
   EOF

* 升级系统::

   apt-get update -y && apt-get upgrade -y && apt-get dist-upgrade -y

* Reboot (内核已经更新)
   
3.2. hostname 设置(根据自己实际情况调整)
----------------------------------------

* 相应文件配置如下::

    # cat /etc/hostname 
    grizzly-controller

    # cat /etc/hosts
    127.0.0.1       localhost
    172.16.9.151    grizzly-controller.hengtiansoft.com     grizzly-controller

    # hostname
    grizzly-controller

    # hostname -f
    grizzly-controller.hengtiansoft.com

   
3.3. 网络设置
--------------

* 保证只有一个网卡能访问Internet ::

    #For Exposing OpenStack API over the internet
    auto eth1
    iface eth1 inet static
            address 172.16.9.151
            netmask 255.255.255.0
            network 172.16.9.0
            broadcast 172.16.9.255
            gateway 172.16.9.254
            # dns-* options are implemented by the resolvconf package, if installed
            dns-nameservers 172.16.5.1
            dns-search hengtiansoft.com


    #Not internet connected(used for OpenStack management)
    auto eth0
        iface eth0 inet static
        address 10.10.10.51
        netmask 255.255.255.0


* 重启网络服务::

   /etc/init.d/networking restart

3.4. MySQL
----------

* 安装 MySQL::

   apt-get install -y mysql-server python-mysqldb

* 允许远程访问MySQL::

   sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/my.cnf
   service mysql restart

* 创建数据库::

   mysql -u root -p
   
   #Keystone
   CREATE DATABASE keystone;
   GRANT ALL ON keystone.* TO 'keystoneUser'@'%' IDENTIFIED BY 'keystonePass';
   GRANT ALL ON keystone.* TO 'keystoneUser'@'localhost' IDENTIFIED BY 'keystonePass';
   
   #Glance
   CREATE DATABASE glance;
   GRANT ALL ON glance.* TO 'glanceUser'@'%' IDENTIFIED BY 'glancePass';
   GRANT ALL ON glance.* TO 'glanceUser'@'localhost' IDENTIFIED BY 'glancePass';

   #Quantum
   CREATE DATABASE quantum;
   GRANT ALL ON quantum.* TO 'quantumUser'@'%' IDENTIFIED BY 'quantumPass';
   GRANT ALL ON quantum.* TO 'quantumUser'@'localhost' IDENTIFIED BY 'quantumPass';

   #Nova
   CREATE DATABASE nova;
   GRANT ALL ON nova.* TO 'novaUser'@'%' IDENTIFIED BY 'novaPass';
   GRANT ALL ON nova.* TO 'novaUser'@'localhost' IDENTIFIED BY 'novaPass';      

   #Cinder
   CREATE DATABASE cinder;
   GRANT ALL ON cinder.* TO 'cinderUser'@'%' IDENTIFIED BY 'cinderPass';
   GRANT ALL ON cinder.* TO 'cinderUser'@'localhost' IDENTIFIED BY 'cinderPass';

   quit;

* 安装 phpMyAdmin(可选) ::

    phpMyAdmin是一个web端的mysql管理工具，安装完成后http://ip/phpmyadmin 就可以管理数据库,
    要比mysql命令行友好方便。

3.5. NTP服务
--------------

* 安装NTP::

   apt-get install -y ntp


3.6. IP转发
--------------

* 修改配置文件::
    
   sed -i -r 's/^\s*#(net\.ipv4\.ip_forward=1.*)/\1/' /etc/sysctl.conf
   sysctl net.ipv4.ip_forward=1

* 检查修改结果::

   # sysctl -p
   net.ipv4.ip_forward = 1


3.7. 其它软件
--------------

* 安装其它相关的软件::

   apt-get install -y vlan bridge-utils

3.8. Keystone
--------------

* 安装Keystone包::

   apt-get install -y keystone

* 编辑配置文件 /etc/keystone/keystone.conf 修改数据库连接::

   connection = mysql://keystoneUser:keystonePass@10.10.10.51/keystone

* 重启Keystone服务，同步数据库::

   service keystone restart
   keystone-manage db_sync

* 用 `Scripts 目录 <https://github.com/mseknibilel/OpenStack-Grizzly-Install-Guide/tree/OVS_MultiNode/KeystoneScripts>` 下的2个脚本来初始化数据库::

   #在执行脚本前，修改变量 **HOST_IP**, **EXT_HOST_IP**, **CINDER_HOST_IP**, **SWIFT_HOST_IP** 为相应的IP
    
   wget https://raw.github.com/mseknibilel/OpenStack-Grizzly-Install-Guide/OVS_MultiNode/KeystoneScripts/keystone_basic.sh
   wget https://raw.github.com/mseknibilel/OpenStack-Grizzly-Install-Guide/OVS_MultiNode/KeystoneScripts/keystone_endpoints_basic.sh
   
   chmod +x keystone_basic.sh
   chmod +x keystone_endpoints_basic.sh
   
   ./keystone_basic.sh
   ./keystone_endpoints_basic.sh

* 创建一个认证文件，然后加载到环境变量::

   nano creds

   #Paste the following:
   export OS_TENANT_NAME=admin
   export OS_USERNAME=admin
   export OS_PASSWORD=admin_pass
   export OS_AUTH_URL="http://172.16.9.151:5000/v2.0/"

   # Load it:
   source creds

* 验证keystone安装是否正确::

   keystone user-list
   keystone role-list
   keystone tenant-list
   keystone endpoint-list
   service keystone status


* Keystone 安装出错时如何解决::

   1. 查看Keystone服务是否正常 service keystone status
   2. 查看 5000 和 35357 端口是否在监听 netstat -ntlp | grep -E "5000|35357"
   3. 查看 /var/log/keystone/keystone.log 报错信息  tail -f -n 400 /var/log/keystone/keystone.log
   4. 2个 shell 脚本执行错误解决：(检查脚本内容变量设置)

   重建KeyStone数据库
   # 如果脚本运行出问题，可以删除数据库，然后重启创建，同步，用修改后的脚本初始化
   
   mysql -uroot -p
   mysql> drop database keystone;
   mysql> create database keystone; 
   mysql> quit;

   keystone-manage db_sync

   然后执行2个 shell 脚本



3.9. Glance
-------------------

  在安装 glance 前，请先到第7节安装 swfit 节点，因为本教程中 glance 用 swift 做后端存储。

* 安装glance包::

   apt-get install -y glance

* 编辑配置文件 /etc/glance/glance-api-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   delay_auth_decision = true
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

* 编辑配置文件 /etc/glance/glance-registry-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

* 编辑配置文件 /etc/glance/glance-api.conf ::

   default_store = swift

   sql_connection = mysql://glanceUser:glancePass@10.10.10.51/glance

   swift_store_auth_version = 2
   swift_store_auth_address = http://172.16.9.151:5000/v2.0/
   swift_store_user = service:glance
   swift_store_key = service_pass
   swift_store_container = glance
   swift_store_create_container_on_put = True

   [paste_deploy]
   flavor = keystone
   
* 编辑配置文件 /etc/glance/glance-registry.conf::

   sql_connection = mysql://glanceUser:glancePass@10.10.10.51/glance

   [paste_deploy]
   flavor = keystone

* 重启 glance-api 和 glance-registry 服务::

   service glance-api restart; service glance-registry restart

* 同步 glance 数据库::

   glance-manage db_sync

* 验证 glance 安装::

   glance image-list

   没有输出表示正常，因为还没有上传 image

* 下载 cirrors image::

   wget https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img

   这个 image 大小只有10M，仅仅作为测试使用。真正要使用请去ubuntu官方下载http://cloud-images.ubuntu.com/，或者自己制作。
   cirrors image的用户名是 cirros， 密码是 cubswin:)

* 上传 cirrors image 到 glance::

   glance image-create --name myFirstImage --is-public true --container-format bare --disk-format qcow2 < ./cirros-0.3.0-x86_64-disk.img

* 现在可以查看上传的 image 了::

   glance image-list

3.10. Quantum
-------------------

* 安装 Quantum server 和 the OpenVSwitch 包::

   apt-get install -y quantum-server

* 编辑 OVS plugin 配置文件 /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini:: 

   #Under the database section
   [DATABASE]
   sql_connection = mysql://quantumUser:quantumPass@10.10.10.51/quantum

   #Under the OVS section
   [OVS]
   tenant_network_type = gre
   tunnel_id_ranges = 1:1000
   enable_tunneling = True

   #Firewall driver for realizing quantum security group function
   [SECURITYGROUP]
   firewall_driver = quantum.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

* 编辑配置文件 /etc/quantum/api-paste.ini ::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass

* 编辑配置文件 /etc/quantum/quantum.conf::

   [keystone_authtoken]
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass
   signing_dir = /var/lib/quantum/keystone-signing

* 重启 quantum server::

   service quantum-server restart


3.11. Nova
-------------------

* 安装 nova 组件::

   apt-get install -y nova-api nova-cert novnc nova-consoleauth nova-scheduler nova-novncproxy nova-doc nova-conductor

* 编辑配置文件 /etc/nova/api-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = nova
   admin_password = service_pass
   signing_dirname = /tmp/keystone-signing-nova
   # Workaround for https://bugs.launchpad.net/nova/+bug/1154809
   auth_version = v2.0

* 编辑配置文件 /etc/nova/nova.conf::

   [DEFAULT] 
   logdir=/var/log/nova
   state_path=/var/lib/nova
   lock_path=/run/lock/nova
   verbose=True
   api_paste_config=/etc/nova/api-paste.ini
   compute_scheduler_driver=nova.scheduler.simple.SimpleScheduler
   rabbit_host=10.10.10.50
   nova_url=http://10.10.10.51:8774/v1.1/
   sql_connection=mysql://novaUser:novaPass@10.10.10.51/nova
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf

   # Auth
   use_deprecated_auth=false
   auth_strategy=keystone

   # Imaging service
   glance_api_servers=10.10.10.51:9292
   image_service=nova.image.glance.GlanceImageService

   # Vnc configuration
   novnc_enabled=true
   novncproxy_base_url=http://172.16.9.151:6080/vnc_auto.html
   novncproxy_port=6080
   vncserver_proxyclient_address=10.10.10.51
   vncserver_listen=0.0.0.0

   # Network settings
   network_api_class=nova.network.quantumv2.api.API
   quantum_url=http://10.10.10.51:9696
   quantum_auth_strategy=keystone
   quantum_admin_tenant_name=service
   quantum_admin_username=quantum
   quantum_admin_password=service_pass
   quantum_admin_auth_url=http://10.10.10.51:35357/v2.0
   libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
   linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
   #If you want Quantum + Nova Security groups
   firewall_driver=nova.virt.firewall.NoopFirewallDriver
   security_group_api=quantum
   #If you want Nova Security groups only, comment the two lines above and uncomment line -1-.
   #-1-firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver

   #Metadata
   service_quantum_metadata_proxy = True
   quantum_metadata_proxy_shared_secret = helloOpenStack

   # Compute #
   compute_driver=libvirt.LibvirtDriver

   # Cinder #
   volume_api_class=nova.volume.cinder.API
   osapi_volume_listen_port=5900

* 同步 nova 数据库::

   nova-manage db sync

* 重启 nova 相关服务::

   cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done ; cd -

* 查看 nova 服务是否正常，笑脸 :-) 表示正常，XXX 表示异常::

   nova-manage service list

3.12. Horizon
--------------

* 安装 horizon 包::

   apt-get install -y openstack-dashboard memcached

* horizon 安装好后，默认的主题是 ubuntu 的，这个主题有问题，删掉::

   dpkg --purge openstack-dashboard-ubuntu-theme 

* 重启 apache 和 memcached 服务::

   service apache2 restart; service memcached restart


4. 网络节点
===============

4.1. 操作系统
-----------------

* 安装ubuntu 12.04.2 Server版本，最小化安装，只需要安装SSH server就可以。安装完成之后切换进入root用户，本文档之后的所有命令都在root下执行::

   sudo su
    
* 添加Grizzly仓库 [只适用于 Ubuntu 12.04]::

   apt-get update
   apt-get install ubuntu-cloud-keyring

   cat <<EOF >>/etc/apt/sources.list
   deb  http://ubuntu-cloud.archive.canonical.com/ubuntu precise-proposed/grizzly main
   deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/grizzly main
   EOF

* 升级系统::

   apt-get update -y && apt-get upgrade -y && apt-get dist-upgrade -y

* Reboot (内核已经更新)
   
4.2. hostname 设置(根据自己实际情况调整)
----------------------------------------

* 相应文件配置如下::

    # cat /etc/hostname 
    grizzly-network

    # cat /etc/hosts
    127.0.0.1       localhost
    10.10.10.52     grizzly-network.hengtiansoft.com        grizzly-network

    # hostname
    grizzly-network

    # hostname -f
    grizzly-network.hengtiansoft.com

   
4.3. 网络设置
--------------

* 保证只有一个网卡能访问Internet::

    # VM internet Access
    auto eth2
    iface eth2 inet static
            address 172.16.9.152
            netmask 255.255.255.0
            network 172.16.9.0
            broadcast 172.16.9.255
            gateway 172.16.9.254
            # dns-* options are implemented by the resolvconf package, if installed
            dns-nameservers 172.16.5.1
            dns-search hengtiansoft.com

    # VM Configuration
    auto eth1
        iface eth1 inet static
        address 10.20.20.52
        netmask 255.255.255.0

    # OpenStack management
    auto eth0
        iface eth0 inet static
        address 10.10.10.52
        netmask 255.255.255.0


* 重启网络服务::

   /etc/init.d/networking restart

4.4. NTP服务
--------------

* 安装NTP::

   apt-get install -y ntp


* 配置 NTP，和 controller 时间同步::
   
   #Comment the ubuntu NTP servers
   sed -i 's/server 0.ubuntu.pool.ntp.org/#server 0.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 1.ubuntu.pool.ntp.org/#server 1.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 2.ubuntu.pool.ntp.org/#server 2.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 3.ubuntu.pool.ntp.org/#server 3.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   
   #Set the compute node to follow up your conroller node
   sed -i 's/server ntp.ubuntu.com/server 10.10.10.51/g' /etc/ntp.conf

   service ntp restart  

4.5. IP转发
--------------

* 修改配置文件::
   
   sed -i -r 's/^\s*#(net\.ipv4\.ip_forward=1.*)/\1/' /etc/sysctl.conf
   sysctl net.ipv4.ip_forward=1

* 检查修改结果::

   # sysctl -p
   net.ipv4.ip_forward = 1

4.6. 其它软件
--------------

* 安装其它相关的软件::

   apt-get install -y vlan bridge-utils


4.7. OpenVSwitch (第一部分)
---------------------------

* 安装 openVSwitch::

   apt-get install -y openvswitch-switch openvswitch-datapath-dkms

* 创建网桥::

   #br-int will be used for VM integration  
   ovs-vsctl add-br br-int

   #br-ex is used to make to VM accessible from the internet
   ovs-vsctl add-br br-ex


4.8. Quantum
------------------

* 安装 Quantum openvswitch agent, l3 agent 和 dhcp agent::

   apt-get -y install quantum-plugin-openvswitch-agent quantum-dhcp-agent quantum-l3-agent quantum-metadata-agent

* 编辑配置文件 /etc/quantum/api-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass

* 编辑配置文件 /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini:: 

   #Under the database section
   [DATABASE]
   sql_connection = mysql://quantumUser:quantumPass@10.10.10.51/quantum

   #Under the OVS section
   [OVS]
   tenant_network_type = gre
   tunnel_id_ranges = 1:1000
   integration_bridge = br-int
   tunnel_bridge = br-tun
   local_ip = 10.20.20.52
   enable_tunneling = True

   #Firewall driver for realizing quantum security group function
   [SECURITYGROUP]
   firewall_driver = quantum.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

* 编辑配置文件 /etc/quantum/metadata_agent.ini::
   
   # The Quantum user information for accessing the Quantum API.
   auth_url = http://10.10.10.51:35357/v2.0
   auth_region = RegionOne
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass

   # IP address used by Nova metadata server
   nova_metadata_ip = 10.10.10.51

   # TCP Port used by Nova metadata server
   nova_metadata_port = 8775

   metadata_proxy_shared_secret = helloOpenStack

* 编辑配置文件 /etc/quantum/quantum.conf ，确保rabbit_host指向rabbitmq节点IP::

   rabbit_host = 10.10.10.51

   #And update the keystone_authtoken section

   [keystone_authtoken]
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass
   signing_dir = /var/lib/quantum/keystone-signing

* 编辑配置文件  /etc/sudoers.d/quantum_sudoers ,给quantum用户所有权限 ::

   nano /etc/sudoers/sudoers.d/quantum_sudoers
   
   #Modify the quantum user
   quantum ALL=NOPASSWD: ALL

* 重启所有 quantum 相关的服务::

   cd /etc/init.d/; for i in $( ls quantum-* ); do sudo service $i restart; done; cd -


4.9. OpenVSwitch (第二部分)
----------------------------
* 编辑网络配置文件 /etc/network/interfaces::

   # VM internet Access
   auto eth2
   iface eth2 inet manual
   up ifconfig $IFACE 0.0.0.0 up
   up ip link set $IFACE promisc on
   down ip link set $IFACE promisc off
   down ifconfig $IFACE down

* 把 eth2 添加到网桥 br-ex 上 ::

   #执行下面命令之后，network 节点将无法访问，因为eth2这个网口配置没了，但是这不影响OpenStack工作
   ovs-vsctl add-port br-ex eth2

   #如果想要网络访问的话, 编辑文件 /etc/network/interfaces，把原来eth2的网络配置，配到br-ex上

    auto br-ex
    iface br-ex inet static
            address 172.16.9.152
            netmask 255.255.255.0
            network 172.16.9.0
            broadcast 172.16.9.255
            gateway 172.16.9.254
            # dns-* options are implemented by the resolvconf package, if installed
            dns-nameservers 172.16.5.1
            dns-search hengtiansoft.com


5. 计算节点
=========================

5.1. 操作系统
-----------------

* 安装ubuntu 12.04.2 Server版本，最小化安装，只需要安装SSH server就可以。安装完成之后切换进入root用户，本文档之后的所有命令都在root下执行::

   sudo su
    
* 添加Grizzly仓库 [只适用于 Ubuntu 12.04]::

   apt-get update
   apt-get install ubuntu-cloud-keyring

   cat <<EOF >>/etc/apt/sources.list
   deb  http://ubuntu-cloud.archive.canonical.com/ubuntu precise-proposed/grizzly main
   deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/grizzly main
   EOF

* 升级系统::

   apt-get update -y && apt-get upgrade -y && apt-get dist-upgrade -y
   
* Reboot (内核已经更新)

5.2. hostname 设置(根据自己实际情况调整)
----------------------------------------

* 相应文件配置如下::

    # cat /etc/hostname 
    grizzly-compute

    # cat /etc/hosts
    127.0.0.1       localhost
    172.16.9.153      grizzly-compute.hengtiansoft.com        grizzly-compute

    # hostname
    grizzly-compute

    # hostname -f
    grizzly-compute.hengtiansoft.com

   
5.3. 网络设置
--------------

* 保证只有一个网卡能访问Internet::

    #For Downloading Deb Packages from Internet repo
    auto eth2
    iface eth2 inet static
        address 172.16.9.153
        netmask 255.255.255.0
        network 172.16.9.0
        broadcast 172.16.9.255
        gateway 172.16.9.254
        # dns-* options are implemented by the resolvconf package, if installed
        dns-nameservers 172.16.5.1
        dns-search hengtiansoft.com

    # VM Configuration
    auto eth1
        iface eth1 inet static
        address 10.20.20.53
        netmask 255.255.255.0

    # OpenStack management
    auto eth0
        iface eth0 inet static
        address 10.10.10.53
        netmask 255.255.255.0


* 重启网络服务::

   service networking restart

5.4. NTP服务
--------------

* 安装NTP服务::

   apt-get install -y ntp


* 配置 NTP，和 controller 时间同步::
   
   #Comment the ubuntu NTP servers
   sed -i 's/server 0.ubuntu.pool.ntp.org/#server 0.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 1.ubuntu.pool.ntp.org/#server 1.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 2.ubuntu.pool.ntp.org/#server 2.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 3.ubuntu.pool.ntp.org/#server 3.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   
   #Set the compute node to follow up your conroller node
   sed -i 's/server ntp.ubuntu.com/server 10.10.10.51/g' /etc/ntp.conf

   service ntp restart  

5.5. IP转发
--------------

* 修改配置文件::
   
   sed -i -r 's/^\s*#(net\.ipv4\.ip_forward=1.*)/\1/' /etc/sysctl.conf
   sysctl net.ipv4.ip_forward=1

* 检查修改结果::

   # sysctl -p
   net.ipv4.ip_forward = 1


5.6. 其它软件
--------------

* 安装其它相关的软件::

   apt-get install -y vlan bridge-utils

5.7 KVM
------------------

* 确保硬件支持虚拟化 ::

   apt-get install -y cpu-checker
   kvm-ok

   如果返回正常，说明硬件虚拟化已经启用，可以继续下面安装KVM；
   如果返回不正常，请检查原因。

* 安装 KVM ::

   apt-get install -y kvm libvirt-bin pm-utils

* 编辑配置文件 /etc/libvirt/qemu.conf 修改 cgroup_device_acl 如下::

   cgroup_device_acl = [
   "/dev/null", "/dev/full", "/dev/zero",
   "/dev/random", "/dev/urandom",
   "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
   "/dev/rtc", "/dev/hpet","/dev/net/tun"
   ]

* 删除默认的虚拟网桥 ::

   virsh net-destroy default
   virsh net-undefine default

* 修改配置文件 /etc/libvirt/libvirtd.conf 启用在线迁移::

   listen_tls = 0
   listen_tcp = 1
   auth_tcp = "none"

* 编辑配置文件 /etc/init/libvirt-bin.conf 修改 libvirtd_opts ::

   env libvirtd_opts="-d -l"

* 编辑文件 /etc/default/libvirt-bin::

   libvirtd_opts="-d -l"

* 重启 libvirt 和 dbus 使配置生效::

    service dbus restart && service libvirt-bin restart

5.8. OpenVSwitch
------------------

* 安装 openVSwitch::

   apt-get install -y openvswitch-switch openvswitch-datapath-dkms

* 创建网桥 ::

   #br-int will be used for VM integration  
   ovs-vsctl add-br br-int

5.9. Quantum
------------------

* 安装 Quantum openvswitch agent::

   apt-get -y install quantum-plugin-openvswitch-agent

* 编辑配置文件 /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini:: 

   #Under the database section
   [DATABASE]
   sql_connection = mysql://quantumUser:quantumPass@10.10.10.51/quantum

   #Under the OVS section
   [OVS]
   tenant_network_type = gre
   tunnel_id_ranges = 1:1000
   integration_bridge = br-int
   tunnel_bridge = br-tun
   local_ip = 10.20.20.53
   enable_tunneling = True
   
   #Firewall driver for realizing quantum security group function
   [SECURITYGROUP]
   firewall_driver = quantum.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

* 编辑配置文件 /etc/quantum/quantum.conf， 注意rabbit_host 设为MQ节点IP::
   
   rabbit_host = 10.10.10.51

   #And update the keystone_authtoken section

   [keystone_authtoken]
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = quantum
   admin_password = service_pass
   signing_dir = /var/lib/quantum/keystone-signing

* 重启服务::

   service quantum-plugin-openvswitch-agent restart

5.10. Nova
------------------

* 安装计算节点nova相关组件::

   apt-get install -y nova-compute-kvm

* 编辑配置文件 /etc/nova/api-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = nova
   admin_password = service_pass
   signing_dirname = /tmp/keystone-signing-nova
   # Workaround for https://bugs.launchpad.net/nova/+bug/1154809
   auth_version = v2.0

* 编辑配置文件 /etc/nova/nova-compute.conf ::
   
   [DEFAULT]
   libvirt_type=kvm
   libvirt_ovs_bridge=br-int
   libvirt_vif_type=ethernet
   libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
   libvirt_use_virtio_for_bridges=True

* 编辑配置文件 /etc/nova/nova.conf::

   [DEFAULT] 
   logdir=/var/log/nova
   state_path=/var/lib/nova
   lock_path=/run/lock/nova
   verbose=True
   api_paste_config=/etc/nova/api-paste.ini
   compute_scheduler_driver=nova.scheduler.simple.SimpleScheduler
   rabbit_host=10.10.10.51
   nova_url=http://10.10.10.51:8774/v1.1/
   sql_connection=mysql://novaUser:novaPass@10.10.10.51/nova
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf

   # Auth
   use_deprecated_auth=false
   auth_strategy=keystone

   # Imaging service
   glance_api_servers=10.10.10.51:9292
   image_service=nova.image.glance.GlanceImageService

   # Vnc configuration
   novnc_enabled=true
   novncproxy_base_url=http://172.16.9.151:6080/vnc_auto.html
   novncproxy_port=6080
   vncserver_proxyclient_address=10.10.10.53
   vncserver_listen=0.0.0.0

   # Network settings
   network_api_class=nova.network.quantumv2.api.API
   quantum_url=http://10.10.10.51:9696
   quantum_auth_strategy=keystone
   quantum_admin_tenant_name=service
   quantum_admin_username=quantum
   quantum_admin_password=service_pass
   quantum_admin_auth_url=http://10.10.10.51:35357/v2.0
   libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
   linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
   #If you want Quantum + Nova Security groups
   firewall_driver=nova.virt.firewall.NoopFirewallDriver
   security_group_api=quantum
   #If you want Nova Security groups only, comment the two lines above and uncomment line -1-.
   #-1-firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
   
   #Metadata
   service_quantum_metadata_proxy = True
   quantum_metadata_proxy_shared_secret = helloOpenStack

   # Compute #
   compute_driver=libvirt.LibvirtDriver

   # Cinder #
   volume_api_class=nova.volume.cinder.API
   osapi_volume_listen_port=5900
   cinder_catalog_info=volume:cinder:internalURL

* 重启所有 nova 服务::

   cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done; cd - 

* 检查nova服务是否正常::

   nova-manage service list


6. 块存储节点
=========================

6.1. 操作系统
-----------------

* 安装ubuntu 12.04.2 Server版本，最小化安装，只需要安装SSH server就可以。安装完成之后切换进入root用户，本文档之后的所有命令都在root下执行::

   sudo su
    
* 添加Grizzly仓库 [只适用于 Ubuntu 12.04]::

   apt-get update
   apt-get install ubuntu-cloud-keyring

   cat <<EOF >>/etc/apt/sources.list
   deb  http://ubuntu-cloud.archive.canonical.com/ubuntu precise-proposed/grizzly main
   deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/grizzly main
   EOF

* 升级系统::

   apt-get update -y && apt-get upgrade -y && apt-get dist-upgrade -y

* Reboot (内核已经更新)
   
6.2. hostname 设置(根据自己实际情况调整)
----------------------------------------

* 相应文件配置如下::

    # cat /etc/hostname 
    grizzly-cinder

    # cat /etc/hosts
    127.0.0.1       localhost
    10.10.10.54     grizzly-cinder.hengtiansoft.com grizzly-cinder


    # hostname
    grizzly-cinder

    # hostname -f
    grizzly-cinder.hengtiansoft.com

   
6.3. 网络设置
--------------

* 保证只有一个网卡能访问Internet::

   #For Downloading Deb Packages from Internet repo
   auto eth1
   iface eth1 inet static
        address 172.16.9.154
        netmask 255.255.255.0
        network 172.16.9.0
        broadcast 172.16.9.255
        gateway 172.16.9.254
        # dns-* options are implemented by the resolvconf package, if installed
        dns-nameservers 172.16.5.1
        dns-search hengtiansoft.com


   #Not internet connected(used for OpenStack management)
   auto eth0
   iface eth0 inet static
   address 10.10.10.54
   netmask 255.255.255.0

* 重启网络服务::

   /etc/init.d/networking restart

6.4. NTP服务
--------------

* 安装NTP::

   apt-get install -y ntp

* 配置 NTP，和 controller 时间同步::
   
   #Comment the ubuntu NTP servers
   sed -i 's/server 0.ubuntu.pool.ntp.org/#server 0.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 1.ubuntu.pool.ntp.org/#server 1.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 2.ubuntu.pool.ntp.org/#server 2.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   sed -i 's/server 3.ubuntu.pool.ntp.org/#server 3.ubuntu.pool.ntp.org/g' /etc/ntp.conf
   
   #Set the compute node to follow up your conroller node
   sed -i 's/server ntp.ubuntu.com/server 10.10.10.51/g' /etc/ntp.conf

   service ntp restart  

6.5. Cinder
--------------

* 安装相关包::

   apt-get install -y cinder-api cinder-scheduler cinder-volume iscsitarget open-iscsi iscsitarget-dkms python-mysqldb

* 配置 iscsi 服务::

   sed -i 's/false/true/g' /etc/default/iscsitarget

* 重启服务::
   
   service iscsitarget start
   service open-iscsi start

* 编辑配置文件 /etc/cinder/api-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   service_protocol = http
   service_host = 172.16.9.151
   service_port = 5000
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = cinder
   admin_password = service_pass
   signing_dir = /var/lib/cinder

* 编辑配置文件 /etc/cinder/cinder.conf::

   [DEFAULT]
   rootwrap_config=/etc/cinder/rootwrap.conf
   sql_connection = mysql://cinderUser:cinderPass@10.10.10.51/cinder
   api_paste_config = /etc/cinder/api-paste.ini
   iscsi_helper=ietadm
   volume_name_template = volume-%s
   volume_group = cinder-volumes
   verbose = True
   auth_strategy = keystone
   iscsi_ip_address=10.10.10.51

* 同步 cinder 数据库::

   cinder-manage db sync

* 创建cinder-volumes卷 ::
  
   这里有2种方式，一种是文件模拟，性能很差，会有问题，一种是硬盘分区。推荐用硬盘分区

   方式1，文件模拟

   dd if=/dev/zero of=cinder-volumes bs=1 count=0 seek=2G
   losetup /dev/loop2 cinder-volumes
   fdisk /dev/loop2
   #Type in the followings:
   n
   p
   1
   ENTER
   ENTER
   t
   8e
   w

   pvcreate /dev/loop2
   vgcreate cinder-volumes /dev/loop2

   方式2，分区，假设可用分区为 /dev/sda6，没有的话自己 fdisk 创建，

   pvcreate /dev/sda6
   vgcreate cinder-volumes /dev/sda6
   

**注意:** 对方式一来说，重启后 volume 会丢失. (点击 `这里 <https://github.com/mseknibilel/OpenStack-Folsom-Install-guide/blob/master/Tricks%26Ideas/load_volume_group_after_system_reboot.rst>`_ 查看如何在重启后加载 volume) 

* 重启 Cinder 服务::

   cd /etc/init.d/; for i in $( ls cinder-* ); do sudo service $i restart; done; cd -

* 查看 Cinder 服务是否正常运行::

   cd /etc/init.d/; for i in $( ls cinder-* ); do sudo service $i status; done; cd -


7. 对象存储节点
==================

7.1. 操作系统
-----------------

* 安装ubuntu 12.04.2 Server版本，最小化安装，只需要安装SSH server就可以。安装完成之后切换进入root用户，本文档之后的所有命令都在root下执行::

   sudo su
    
* 添加Grizzly仓库 [只适用于 Ubuntu 12.04]::

   apt-get update
   apt-get install ubuntu-cloud-keyring

   cat <<EOF >>/etc/apt/sources.list
   deb  http://ubuntu-cloud.archive.canonical.com/ubuntu precise-proposed/grizzly main
   deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/grizzly main
   EOF

* 升级系统::

   apt-get update -y && apt-get upgrade -y && apt-get dist-upgrade -y
   
* Reboot (内核已经更新)

7.2. hostname 设置(根据自己实际情况调整)
----------------------------------------

* 相应文件配置如下::

    # cat /etc/hostname 
    grizzly-swift

    # cat /etc/hosts
    127.0.0.1       localhost
    10.10.10.55     grizzly-swift.hengtiansoft.com  grizzly-swift

    # hostname
    grizzly-swift

    # hostname -f
    grizzly-swift.hengtiansoft.com

   
7.3. 网络设置
--------------

* Only one NIC should have an internet access::

    #For Downloading Deb Packages from Internet repo
    auto eth1
    iface eth1 inet static
        address 172.16.9.155
        netmask 255.255.255.0
        network 172.16.9.0
        broadcast 172.16.9.255
        gateway 172.16.9.254
        # dns-* options are implemented by the resolvconf package, if installed
        dns-nameservers 172.16.5.1
        dns-search hengtiansoft.com


    #Not internet connected(used for OpenStack management)
    auto eth0
        iface eth0 inet static
        address 10.10.10.55
        netmask 255.255.255.0


* 重启网络服务::

   /etc/init.d/networking restart

7.4. swift
--------------

* 安装 swift 相关软件::

   apt-get -y install swift swift-proxy swift-account swift-container swift-object xfsprogs curl python-pastedeploy python-webob


* 分区::
   
   假设可用分区为 /dev/sda6，没有的话自己 fdisk 创建。

   格式化分区
   mkfs.xfs -f -i size=1024 /dev/sdb1

   创建挂载点

   mkdir /mnt/swift_backend

   修改/etc/fstab, 加上
   /dev/sda6 /mnt/swift_backend xfs noatime,nodiratime,nobarrier,logbufs=8 0 0

   检查修改是否正确

   mount -a
   如果fstab有错误，会进行提示。没错误，就会把目录挂载上。

* 目录设置::

   pushd /mnt/swift_backend
   mkdir node1 node2 node3 node4
   popd
   chown swift.swift /mnt/swift_backend/*
   for i in {1..4}; do sudo ln -s /mnt/swift_backend/node$i /srv/node$i; done;
   mkdir -p /etc/swift/account-server \
   /etc/swift/container-server \
   /etc/swift/object-server \
   /srv/node1/device \
   /srv/node2/device \
   /srv/node3/device \
   /srv/node4/device
   mkdir /run/swift
   chown -L -R swift.swift /etc/swift /srv/node[1-4]/ /run/swift

   为了在系统启动时启动Swift服务，需要把如下两行命令写入 /etc/rc.local里，位置在“exit 0;”之前：

   sudo mkdir /run/swift
   sudo chown swift.swift /run/swift

* 配置rsync ::

   编辑 /etc/default/rsync文件
   sed -i 's/RSYNC_ENABLE=false/RSYNC_ENABLE=true/g' /etc/default/rsync

   创建 /etc/rsyncd.conf
    cat > /etc/rsyncd.conf <<EOF
    # General stuff
    uid = swift
    gid = swift
    log file = /var/log/rsyncd.log
    pid file = /run/rsyncd.pid
    address = 127.0.0.1

    # Account Server replication settings
    [account6012]
    max connections = 25
    path = /srv/node1/
    read only = false
    lock file = /run/lock/account6012.lock

    [account6022]
    max connections = 25
    path = /srv/node2/
    read only = false
    lock file = /run/lock/account6022.lock

    [account6032]
    max connections = 25
    path = /srv/node3/
    read only = false
    lock file = /run/lock/account6032.lock

    [account6042]
    max connections = 25
    path = /srv/node4/
    read only = false
    lock file = /run/lock/account6042.lock

    # Container server replication settings

    [container6011]
    max connections = 25
    path = /srv/node1/
    read only = false
    lock file = /run/lock/container6011.lock

    [container6021]
    max connections = 25
    path = /srv/node2/
    read only = false
    lock file = /run/lock/container6021.lock

    [container6031]
    max connections = 25
    path = /srv/node3/
    read only = false
    lock file = /run/lock/container6031.lock

    [container6041]
    max connections = 25
    path = /srv/node4/
    read only = false
    lock file = /run/lock/container6041.lock

    # Object Server replication settings

    [object6010]
    max connections = 25
    path = /srv/node1/
    read only = false
    lock file = /run/lock/object6010.lock

    [object6020]
    max connections = 25
    path = /srv/node2/
    read only = false
    lock file = /run/lock/object6020.lock

    [object6030]
    max connections = 25
    path = /srv/node3/
    read only = false
    lock file = /run/lock/object6030.lock

    [object6040]
    max connections = 25
    path = /srv/node4/
    read only = false
    lock file = /run/lock/object6040.lock
    EOF


    重启rsync服务

    service rsync restart

* Swift配置文件::

    cat >/etc/swift/swift.conf <<EOF
    [swift-hash]
    # random unique string that can never change (DO NOT LOSE)
    swift_hash_path_suffix = `od -t x8 -N 8 -A n </dev/random`
    EOF

* proxy Server::

    创建 /etc/swift/proxy-server.conf

    cat > /etc/swift/proxy-server.conf <<EOF
    [DEFAULT]
    #bind_ip = 10.10.10.56
    bind_port = 8888
    user = swift
    signing_dir = /var/cache/swift
    log_level = DEBUG

    [pipeline:main]
    pipeline = healthcheck cache authtoken keystoneauth proxy-server

    [app:proxy-server]
    use = egg:swift#proxy
    allow_account_management = true
    account_autocreate = true

    [filter:keystoneauth]
    use = egg:swift#keystoneauth
    operator_roles = Member,admin,swiftoperator
    #operator_roles = admin
    #is_admin = false

    [filter:authtoken]
    paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory

    # Delaying the auth decision is required to support token-less
    # usage for anonymous referrers ('.r:*').
    delay_auth_decision = true

    # cache directory for signing certificate
    # signing_dir = /home/swift/keystone-signing

    # auth_* settings refer to the Keystone server
    auth_protocol = http
    auth_host = 172.16.9.151
    auth_port = 35357
    service_port = 5000
    service_host = 172.16.9.151

    # the same admin_token as provided in keystone.conf
    #admin_token = 012345SECRET99TOKEN012345

    # the service tenant and swift userid and password created in Keystone
    admin_tenant_name = service
    admin_user = swift
    admin_password = service_pass

    [filter:cache]
    use = egg:swift#memcache

    [filter:catch_errors]
    use = egg:swift#catch_errors

    [filter:healthcheck]
    use = egg:swift#healthcheck
    EOF

* Account Server, Container Server, Object Server ::

    for x in {1..4}; do
    cat > /etc/swift/account-server/$x.conf <<EOF
    [DEFAULT]
    devices = /srv/node$x
    mount_check = false
    bind_port = 60${x}2
    user = swift
    log_facility = LOG_LOCAL2
     
    [pipeline:main]
    pipeline = account-server

    [app:account-server]
    use = egg:swift#account
     
    [account-replicator]
    vm_test_mode = no

    [account-auditor]

    [account-reaper]
    EOF

    cat >/etc/swift/container-server/$x.conf <<EOF
    [DEFAULT]
    devices = /srv/node$x
    mount_check = false
    bind_ip = 0.0.0.0
    bind_port = 60${x}1
    user = swift
    log_facility = LOG_LOCAL2

    [pipeline:main]
    pipeline = container-server

    [app:container-server]
    use = egg:swift#container

    [container-replicator]
    vm_test_mode = no

    [container-updater]

    [container-auditor]

    [container-sync]
    EOF
     

    cat > /etc/swift/object-server/${x}.conf <<EOF
    [DEFAULT]
    devices = /srv/node${x}
    mount_check = false
    bind_port = 60${x}0
    user = swift
    log_facility = LOG_LOCAL2

    [pipeline:main]
    pipeline = object-server

    [app:object-server]
    use = egg:swift#object

    [object-replicator]
    vm_test_mode = no

    [object-updater]

    [object-auditor]
    EOF


    cat <<EOF >>/etc/swift/container-server.conf 
    [container-sync]
    EOF
    done


    设置日志

    sed -i 's/LOCAL2/LOCAL3/g' /etc/swift/account-server/2.conf
    sed -i 's/LOCAL2/LOCAL4/g' /etc/swift/account-server/3.conf
    sed -i 's/LOCAL2/LOCAL5/g' /etc/swift/account-server/4.conf
    sed -i 's/LOCAL2/LOCAL3/g' /etc/swift/container-server/2.conf
    sed -i 's/LOCAL2/LOCAL4/g' /etc/swift/container-server/3.conf
    sed -i 's/LOCAL2/LOCAL5/g' /etc/swift/container-server/4.conf
    sed -i 's/LOCAL2/LOCAL3/g' /etc/swift/object-server/2.conf
    sed -i 's/LOCAL2/LOCAL4/g' /etc/swift/object-server/3.conf
    sed -i 's/LOCAL2/LOCAL5/g' /etc/swift/object-server/4.conf

    Ring Server

    pushd /etc/swift
    swift-ring-builder object.builder create 18 3 1
    swift-ring-builder container.builder create 18 3 1
    swift-ring-builder account.builder create 18 3 1
    swift-ring-builder object.builder add z1-127.0.0.1:6010/device 1
    swift-ring-builder object.builder add z2-127.0.0.1:6020/device 1
    swift-ring-builder object.builder add z3-127.0.0.1:6030/device 1
    swift-ring-builder object.builder add z4-127.0.0.1:6040/device 1
    swift-ring-builder object.builder rebalance
    swift-ring-builder container.builder add z1-127.0.0.1:6011/device 1
    swift-ring-builder container.builder add z2-127.0.0.1:6021/device 1
    swift-ring-builder container.builder add z3-127.0.0.1:6031/device 1
    swift-ring-builder container.builder add z4-127.0.0.1:6041/device 1
    swift-ring-builder container.builder rebalance
    swift-ring-builder account.builder add z1-127.0.0.1:6012/device 1
    swift-ring-builder account.builder add z2-127.0.0.1:6022/device 1
    swift-ring-builder account.builder add z3-127.0.0.1:6032/device 1
    swift-ring-builder account.builder add z4-127.0.0.1:6042/device 1
    swift-ring-builder account.builder rebalance

* 启动相关服务::
    
    设置目录权限
    chown -R swift.swift /etc/swift

    启动swift服务
    swift-init main start
    swift-init rest start


    验证

    swift -v -V 2.0 -A http://172.16.9.151:5000/v2.0/ -U service:glance -K service_pass stat

    StorageURL: http://172.16.9.155:8888/v1/AUTH_25e6b21a72a04cbc8ea856139562b27a
    Auth Token: 3f85c92d6860444e90bf0e1bedc4b45a
       Account: AUTH_25e6b21a72a04cbc8ea856139562b27a
    Containers: 0
       Objects: 0
         Bytes: 0
    Accept-Ranges: bytes
    X-Trans-Id: txea28887460ff4f1d84e9e826e5514711


    在glance安装完成后，可以查看上传的镜像
    swift -v -V 2.0 -A http://172.16.9.151:5000/v2.0/ -U service:glance -K service_pass list glance


8. 创建虚拟机
================

在创建虚拟机前，我们先要创建 tenant, user 和 内部网络

* 创建一个 tenant ::

   keystone tenant-create --name project_one

* 创建一个新用户，并且把赋新用户的角色为tenant项目下的memeber角色(keystone role-list 可以得到member角色的id)::

   keystone user-create --name=user_one --pass=user_one --tenant-id $put_id_of_project_one --email=user_one@domain.com
   keystone user-role-add --tenant-id $put_id_of_project_one  --user-id $put_id_of_user_one --role-id $put_id_of_member_role

* 给新的 tenant 创建一个网络::

   quantum net-create --tenant-id $put_id_of_project_one net_proj_one 

* 在tenant网络下面创建一个子网::

   quantum subnet-create --tenant-id $put_id_of_project_one net_proj_one 50.50.1.0/24

* 给tenant创建一个路由router::

   quantum router-create --tenant-id $put_id_of_project_one router_proj_one

* 把router加到l3 agent上面（如果没有自动添加的话）::

   quantum agent-list (to get the l3 agent ID)
   quantum l3-agent-router-add $l3_agent_ID router_proj_one

* 把router添加到子网上面::

   quantum router-interface-add $put_router_proj_one_id_here $put_subnet_id_here

* 重启所有 quantum 服务::

   cd /etc/init.d/; for i in $( ls quantum-* ); do sudo service $i restart; done

* 创建一个外部网络，tenant id 用admin的tenant(keystone tenant-list 得到 admin tenant的id)::

   quantum net-create --tenant-id $put_id_of_admin_tenant ext_net --router:external=True

* 创建一个子网，用来分配floating ip::

   quantum subnet-create --tenant-id $put_id_of_admin_tenant --allocation-pool start=172.16.9.110,end=172.16.9.149 --gateway 172.16.9.254 ext_net 172.16.9.0/24 --enable_dhcp=False --dns_nameservers list=true 172.16.5.1 172.16.5.2

* 把router的网状设成外网:: 

   quantum router-gateway-set $put_router_proj_one_id_here $put_id_of_ext_net_here

* 创建对应于 project one 的 creds 文件并加载::

   nano creds_proj_one

   #Paste the following:
   export OS_TENANT_NAME=project_one
   export OS_USERNAME=user_one
   export OS_PASSWORD=user_one
   export OS_AUTH_URL="http://172.16.9.151:5000/v2.0/"

   source creds_proj_one

* 添加安全规则，打开icmp，使虚拟机能够ping通，打开ssh 22端口::

   nova --no-cache secgroup-add-rule default icmp -1 -1 0.0.0.0/0
   nova --no-cache secgroup-add-rule default tcp 22 22 0.0.0.0/0

* 给project one 项目分配一个floating ip::

   quantum floatingip-create ext_net

* 创建一个虚拟机::

   nova --no-cache boot --image $id_myFirstImage --flavor 1 my_first_vm 

* 找到对应于刚创建的虚拟机的port::

   quantum port-list

* 把floating ip 绑定的新创建的虚拟机上::

   quantum floatingip-associate $put_id_floating_ip $put_id_vm_port

好了，现在就可以 ping 刚创建的虚拟机，ssh登录，开始使用 OpenStack 吧。 :-)


9. 联系方式
===========

Fungo Wang  : bo@fungo.me
http://fungo.me/

10. 致谢
=================

本教程基于:

* Bilel Msekni's Grizzly Install guide [https://github.com/mseknibilel/OpenStack-Grizzly-Install-Guide]
* Ubuntu 12.04 Openstack Essex 安装（单节点）Swift篇 [http://www.chenshake.com/swift-single-version/#GlanceSwift]
* OpenStack Grizzly Swift 单节点安装 [http://longgeek.com/2013/04/19/openstack-grizzly-swift-single-node-installation/]