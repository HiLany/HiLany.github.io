---
layout:     post
title:      Greenplum CommandCenter安装
subtitle:   GreenplumCCWeb的介绍以及安装步骤
date:       2019-02-16
author:     LANY
header-img: img/post-20190217-bg.jpg
catalog: true
tags:
    - Pivotal
    - Greenplum
    - MPP
    - 大数据
    - 实时监控
---
# Greenplum CommandCenter是什么?


Greenplum CommandCenter是Pivotal Greenplum Database大数据平台的一个网页版管理工具。

Greenplum CommandCenter监控其平台的性能指标，分析其集群的健康状况并且使数据库管理员能够在Greenplum数据库环境中执行管理任务。


# Greenplum CommandCenter监控安装过程

本次安装的CC-Web版本为 ***3.0.2***。

## 1.安装前准备

- Greenplum安装完成且集群为启动状态

- 下载Greenplum CommandCenter安装包

CCWeb安装包需要在[Pivotal官网](https://network.pivotal.io/products/pivotal-gpdb/#/releases/2684)进行下载。

- 设置postgresql.conf文件，增加启用监控的参数。（这些参数默认会添加在文件的末尾）

        gp_enable_gpperfmon=on
        gpperfmon_port=8888
        gp_external_enable_exec=on
        gpperfmon_log_alert_level=warning
        
## 2.安装gp监控的数据库和用户等信息

- performance monitor安装
       
        $ gpperfmon_install --enable --password gpadmin --port 5432

- 重启数据库

		$ gpstop -r
		
- 检查gp监控是否启动

		$ ps -ef | grep gpmmon
		
- 检查gp监控是否检测到每台主机
		
		$ psql -d 'gpperfmon' -c 'select * from system_now;'
 

## 3.正式安装greenplum-cc-web
- 在root下安装greenplum-cc-web
		
		
		$ exit(gpadmin用户下)
		# ./greenplum-cc-web-3.0.2-LINUX-x86_64.bin 
		
- 配置root下的.bashrc，在./bashrc中添加下面的环境变量
 		
 		source /usr/local/greenplum-cc-web/gpcc_path.sh 
		source /usr/local/greenplum-db-4.3.12.0/greenplum_path.sh
 		
- 使环境变量生效

		# source ~/.bashrc

- 将安装的cc-web授予gpadmin用户权限

		# chown -R gpadmin /usr/local/greenplum-cc-web
		# chown -R gpadmin /usr/local/greenplum-cc-web-3.0.2
		
- 配置gpadmin下的./bashrc，在./bashrc中添加下面的环境变量

		source /usr/local/greenplum-cc-web/gpcc_path.sh 
		source /usr/local/greenplum-db-4.3.12.0/greenplum_path.sh
		export MASTER_DATA_DIRECTORY=/data/master/gpseg-1
		
- 将gpadmin下的./bashrc，分配到除master之外的主机上

		$ gpscp -f all_segs ~/.bashrc =:~
		
- 用root用户，在除master之外的主机上安装greenplum-cc-web

		$ exit(gpamdin用户下)
		# gpccinstall -f all_segs
		
- 安装完成之后,将安装的文件授予gpadmin用户权限，并生效./bashrc文件

		# gpssh -f all_segs -e 'chown -R gpadmin /usr/local/greenplum-cc-web'
		# gpssh -f all_segs -e 'chown -R gpadmin /usr/local/greenplum-cc-web-3.0.2'
		# su gpadmin
		$ gpssh -f all_segs -e 'source ~/.bashrc'
		
- 在pg_hbc_conf中添加用户登录权限（如果不添加可能会不能创建gpcc的实例）

		host  all   all   ::1/128  trust
		
- 配置gpcc实例

		$ gpcmdr --setup
		
- 启动gpcc实例

		$ gpcmdr --start [your instance name]