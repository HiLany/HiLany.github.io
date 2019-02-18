---
layout:     post
title:      Greenplum4.3.12安装
subtitle:   Greenplum大数据平台安装配置手册
date:       2019-02-17
author:     LANY
header-img: img/post-20190217-bg.png
catalog: true
tags:
    - Pivotal
    - Greenplum
    - MPP
    - 大数据
    - 数据仓库
---
#  Greenplum安装配置手册
----------
## 一、系统要求
- 操作系统
 - SUSE Linux Enterprise Server 11 SP2
 - CentOS 5.0 +
 - Red Hat Enterprise Linux (RHEL)+
 - Oracle Unbreakable Linux 5.5
- 文件系统
 - xfs required for data storage on SUSE Linux and Red Hat (ext3 supported for root file system)
- 内存
 - 16 GB RAM per server
- 硬盘
 - 150MB per host for Greenplum installation
 - Approximately 300MB per segment instance for
meta data
 - Appropriate free space for data with disks at no
more than 70% capacity
 - High-speed, local storage
- 网络
 - 10 Gigabit Ethernet within the array
Dedicated, non-blocking switch
 - 16 GB RAM per server
- 软件和工具
 - bash shell
 - GNU tars
 - GNU zip
 - GNU sed (used by Greenplum Database gpinitsystem)

## 二、安装文件
- greenplum-db-4.3.12.0-rhel5-x86_64.zip
- greenplum-cc-web-3.0.2-LINUX-x86_64.zip
- madlib-ossv1.10.0_pv1.9.7_gpdb4.3orca-rhel5-x86_64.tgz
- plperl-ossv5.12.4_pv1.3_gpdb4.3orca-rhel7-x86_64.gppkg
- postgis-ossv2.0.3_pv2.0.1_gpdb4.3orca-rhel5-x86_64.gppkg
 
## 三、安装过程
以下安装过程使用九台主机机，在RHEL7.3（最小安装，中文字符集）操作系统中完成。
集群清单:
Master：  10.10.10.1
Segment1： 10.10.10.2
Segment2： 10.10.10.3
Segment3： 10.10.10.4
Segment4： 10.10.10.5
Segment5： 10.10.10.6
Segment6： 10.10.10.7
Segment7： 10.10.10.8
Segment8： 10.10.10.9

**1.安装前系统相关设置**

以下操作使用root用户在所有主机上都进行设置
（1） 关闭Selinux。

```bash
# su - root
# vi /etc/selinux/config
```
修改以下内容:

```
SELINUX=disabled
```
（2） 关闭防火墙

```bash
# systemctl stop firewalld
# systemctl disable firewalld
```
（3） 设置系统参数

```bash
# vi /etc/sysctl.conf
```
新增以下内容:

```
kernel.shmmax = 500000000
kernel.shmmni = 4096
kernel.shmall = 4000000000
kernel.sem = 250 512000 100 2048
kernel.sysrq = 1
kernel.core_uses_pid = 1
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.msgmni = 2048
net.ipv4.tcp_syncookies = 1
net.ipv4.ip_forward = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.conf.all.arp_filter = 1
net.ipv4.ip_local_port_range = 1025 65535
net.core.netdev_max_backlog = 10000
net.core.rmem_max = 2097152
net.core.wmem_max = 2097152
vm.overcommit_memory = 2
```
（4） 设置系统安全限制参数

```bash
# vi /etc/security/limits.conf
```
新增以下内容：

```
* soft nofile 65536
* hard nofile 65536
* soft nproc 131072
* hard nproc 131072
```
（5） 设置XFS参数

```bash
# vi /etc/fstab
```
在xfs格式的配置中修改default为以下内容：

```
rw,nodev,noatime,attr2,nobarrier,inode64,allocsize=16384k,noquota
```

（6） Linux磁盘IO设置

```bash
# grubby --update-kernel=ALL --args="elevator=deadline"
```
（7） 设置blockdev

```bash
# echo '/sbin/blockdev --setra 16384 /dev/sda' >> /etc/rc.local
# echo '/sbin/blockdev --setra 16384 /dev/sdb' >> /etc/rc.local
# chmod +x /etc/rc.d/rc.local
```
（8） 分别设置主机名

```bash
# hostnamectl  --static  set-hostname  gpmaster
# hostnamectl  --static  set-hostname  gpseg1
# hostnamectl  --static  set-hostname  gpseg2
# hostnamectl  --static  set-hostname  gpseg3
# hostnamectl  --static  set-hostname  gpseg4

```
（9） 创建用户gpadmin

```bash
# useradd -r -m gpadmin
# passwd gpadmin 
```
（10） 关闭IPC

```bash
# vi /etc/systemd/logind.conf
```
修改以下内容，去掉#注释：

```
RemoveIPC=no
```
重启服务

```bash
# systemctl restart  systemd-logind.service
```

（11）修改hosts文件

```bash
#  vi /etc/hosts
```
修改以下内容：

```
10.10.10.1 gpmaster
10.10.10.2 gpseg1
10.10.10.3 gpseg2
10.10.10.4 gpseg3
10.10.10.5 gpseg4
```
（12）NTP设置
RHEL7.3默认使用的是chrony。
修改各个standby和segment的主机相应配置,将NTP服务器指向gpmaster。

安装chrony-2.1.1-3.el7.x86_64.rpm

```bash
rpm  -ivh  chrony-2.1.1-3.el7.x86_64.rpm
# vim /etc/chrony.conf
```
修改内容：

```
server 10.10.10.26 iburst
# iburst ?
```
重启chrony

```bash
systemctl enable chronyd
systemctl restart chronyd
```
查看chronyc状态

```bash
# chronyc sources
```

(13) 安装缺失的rpm包
master上安装unzip.rpm

```bash
rpm -ivh unzip-6.0-16.el7.x86_64.rpm
```

**2.数据库安装**

创建用户gpadmin，在gpadmin的当前目录中创建gp文件夹，将安装文件复制到该文件夹。
（1）创建主机清单文件
所有主机清单：

```bash
# cd /home/gpadmin/gp
# vi all_hosts
```
新增以下内容：

```
gpmaster
gpseg1
gpseg2
gpseg3
gpseg4
gpseg5
gpseg6
gpseg7
gpseg8
```
除master之外所有主机清单：

```bash
# vi all_std_segs
```
新增以下内容：

```
gpseg1
gpseg2
gpseg3
gpseg4
gpseg5
gpseg6
gpseg7
gpseg8
```

所有计算节点主机清单

```bash
# vi all_segs
```
新增以下内容：

```
gpseg1
gpseg2
gpseg3
gpseg4
gpseg5
gpseg6
gpseg7
gpseg8
```
（2）在Master上安装GP

```bash
# su - root
# unzip greenplum-db-4.3.12.0-rhel5-x86_64.zip
# /bin/bash greenplum-db-4.3.12.0-rhel5-x86_64.bin 
```
（3）主机互信 (授予gpadmin  greenplum-db  chown -R  gpadmin greenplum-db)

```bash
# source /usr/local/greenplum-db/greenplum_path.sh
# gpssh-exkeys -f all_hosts
 
```
（3.1）在其它主机上远程安装net-tools

```bash
#  gpssh -f all_std_segs -e 'rpm -ivh /root/net-tools-2.0-0.17.20131004git.el7.x86_64.rpm'
```
（4）在除master之外的主机上安装GP

```bash
# gpseginstall -f all_std_segs -u gpadmin -p passwd
```
（5）验证安装

```bash
$ su - gpadmin
$ source /usr/local/greenplum-db/greenplum_path.sh
$ gpssh -f all_hosts -e ls -l $GPHOME
成功之后，个主机之间可ssh无密码登录，若不行，则重新执行命令
$ gpssh-exkeys -f all_hosts
```
（6）创建目录
在master上创建存储目录

```bash
# su - root
#  mkdir /data/master
# chown -R gpadmin /data/master
# source /usr/local/greenplum-db/greenplum_path.sh
```
在segment上创建存储目录

```bash
#  source /usr/local/greenplum-db/greenplum_path.sh
#  gpssh -f all_segs -e 'mkdir /data/primary1'
#  gpssh -f all_segs -e 'mkdir /data/primary2'
#  gpssh -f all_segs -e 'mkdir /data/primary3'
#  gpssh -f all_segs -e 'mkdir /data/primary4'
#  gpssh -f all_segs -e 'mkdir /data/primary5'
#  gpssh -f all_segs -e 'mkdir /data/primary6'
#  gpssh -f all_segs -e 'mkdir /data/primary7'
#  gpssh -f all_segs -e 'mkdir /data/primary8'

#  gpssh -f all_segs -e 'chown gpadmin /data/primary1'
#  gpssh -f all_segs -e 'chown gpadmin /data/primary2'
#  gpssh -f all_segs -e 'chown gpadmin /data/primary3'
#  gpssh -f all_segs -e 'chown gpadmin /data/primary4'
#  gpssh -f all_segs -e 'chown gpadmin /data/primary5'
#  gpssh -f all_segs -e 'chown gpadmin /data/primary6'
#  gpssh -f all_segs -e 'chown gpadmin /data/primary7'
#  gpssh -f all_segs -e 'chown gpadmin /data/primary8'
```
（7）oracle兼容（略过）


（8）验证
验证操作系统设置

```bash
# su - gpadmin
# source /usr/local/greenplum-db/greenplum_path.sh
# cp $GPHOME/etc/gpcheck.cnf  ~/gp
# vi gpcheck.cnf 
```
修改第一行

```
rw,nodev,noatime,attr2,nobarrier,inode64,allocsize=16384k,noquota
```
执行以下命令

```bash
# gpcheck -f all_hosts -m gpmaster -s gpstandby --config gpcheck.cnf 
```
验证磁盘I/O和内存带宽（略过）

```bash
# gpcheckperf -f all_segs -r ds -D  -d /data/primary1 -d /data/primary2 -d /data/mirror1 -d /data/mirror2

```
（9）初始化数据库
复制GP安装目录中gpinit模板文件进行修改。

```bash
$ su - gpadmin
$ cp $GPHOME/docs/cli_help/gpconfigs/gpinitsystem_config /home/gpadmin/gpconfigs/gpinitsystem_config
```
修改内容

```
ARRAY_NAME="GP Test"
SEG_PREFIX=gpseg
PORT_BASE=40000
declare -a DATA_DIRECTORY=(/data/primary1 /data/primary2 /data/primary3 /data/primary4 data/primary5 data/primary6 data/primary7 data/primary8)
MASTER_HOSTNAME=gpmaster
MASTER_DIRECTORY=/data/master
MASTER_PORT=5432
TRUSTED SHELL=ssh
CHECK_POINT_SEGMENTS=8
ENCODING=UNICODE
```
执行初始化

```bash
$ gpinitsystem -c gpinit_conf -h all_segs -s standby_master_hostname 
```
（10）配置环境变量

```bash
$  vi ~/.bashrc
```

修改如下：

```
source /usr/local/greenplum-db/greenplum_path.sh
export MASTER_DATA_DIRECTORY=/data/master/gpseg-1
```

**3.使用**

（1）创建数据库

```bash
createdb testdb
psql -d testdb
cd /data/master/gpseg-1
vi pg_hba.conf 
```
（2）修改 允许任意主机访问

```
host     all         all             0.0.0.0/0     trust
指定用户以及规定的IP 才能访问
host     all         gpadmin        10.10.10.0/24      trust
```

（3）加载修改的文件

```bash
gpstop -u
```
