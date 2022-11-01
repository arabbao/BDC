# 数据同步

[TOC]

## 概述

数据采集作为 Inventec 大数据系统体系的第一环尤为重要。因此 BDC 团队 建立了一套标准的数据采集方案，致力规范、高性能地完成海量数据的采集，并将其传输到大数据平台。

我们将数据采集分为日志采集和数据库数据同步两部分。本章侧重讲解数据从业务系统同步进入数据仓库这个环节，但其适用性并不仅限于此。

## 数据同步基础

源业务系统的数据类型多种多样，有来自于关系型数据库的结构化数据，如 Oracle、SQL Server、 MySQL 等; 也有来源于非关系型数据库的非结构化数据，如 MongoDB 等了; 还有来源于文件系统的结构化或非结构化数据，如文件存储 NAS，Samba，对象存储 Minio，OSS 等，这类数据通常以文件形式存储。

数据同步需要针对不同的数据类型及业务场景选择不同的同步方式。总的来说，同步方式可以分为三类: 数据库直连同步、数据库日志解析同步、数据文件同步。

### 数据库直连同步

### 数据库日志解析同步

### 数据文件同步

## Inventec数仓的同步方式

### 数据库直连同步

对接 MES / WMS 系统数据库采用了 DB 数据同步方式，实现以表为粒度进行同步。

#### 作业流程

![](http://assets.processon.com/chart_image/635f971c5653bb070e224299.png)

#### 前置条件

- 暂不支持 blob 字段类型数据同步
- 若需`增量同步`，表必须有一个自增字段，例如 UDT，当更新某一行记录的时候，该字段都会同步更新
- 若需`增量合并`，表除了要包含自增字段，表中每条记录需要有一个唯一标识，可以是主键，唯一索引

#### 具体操作

1. 拷贝模板任务

```bash
git clone https://az-gitlab.inventectj.com/bdcc/chicony-power-datajobs.git
cd chicony-power-datajobs
cp -r job_data_sync_template job_data_sync_demo
```

2. 修改 config.properties 文件

```properties
// 源端数据库实例唯一标识
SRC_TYPE=<JOB_ID>

// JDBC配置 -- 源端为 Oracle
SOURCE_DB_URL=jdbc:oracle:thin:@//<HOST>:<PORT>
SOURCE_DB_CLASSNAME=oracle.jdbc.driver.OracleDriver
SOURCE_DB_USER=<USERNAME>
SOURCE_DB_PASSWORD=<PASSWORD>

// JDBC配置 -- 目标端为 PostgreSQL/Greenplum
TARGET_DB_URL=jdbc:postgresql://<HOST>:<PORT>/<DATABASE>
TARGET_DB_CLASSNAME=org.postgresql.Driver
TARGET_DB_USER=<USERNAME>
TARGET_DB_PASSWORD=<PASSWORD>
```

3. 提交任务配置到配置中心(OSS)

```bash
cd chicony-power-datajobs
./sync.sh
```

4. Infra Monitor中配置定时任务

![](待完善截图)

5. 明确待同步的表，字段，以及表同步规则

   主要弄明白以下四个问题:

   - 源表同步方式:  要同步哪张表，哪些字段，是选择`全量同步`/`增量追加`/`增量合并`中哪一类
   - 增量同步字段:  如果同步方式选择`增量追加`/`增量合并`，就需确定哪个字段可以作为自增字段
   - 主键/业务主键: 如果同步方式选择了`增量合并`，还需确定用哪几个字段来做表关联更新时的关联条件
   - 源表读取约束:  如果初始同步的数据量较大，超过4kw，最好自定义源表查询条件，避免单批次数据量太大出现同步超时

6. 目标端数据库新建待同步表，以及主键，唯一索引

```sql
-- 构造 DDL 语句
-- 此处省略 ...
```

7. 访问目标端数据库中，在数据同步配置表中添加待同步表

```sql
-- 全量同步
INSERT INTO manager.job_data_sync(
	src_schema_name, src_table_name,
	dst_schema_name, dst_table_name, 
	src_select_statement, 
	sync_mode, 
	fields_mapping, 
	inuse,
	src_type, src_db_name,
	src_where_statement 
	)
	VALUES (
	'SAJET', 'SYS_TERMINAL',
	'sajet', 'sys_terminal',
	'CONTROL_ID, TERMINAL_ID, TERMINAL_NAME, PDLINE_ID, PROCESS_ID, STAGE_ID, UPDATE_USERID, UPDATE_TIME, ENABLED, TGS_CONTROL, MACHINE_LINK, MACHINE_ID, SMT_SIDE_TYPE',
	'full', 
	'{}', 
	true,
	'CHICONY', 'MESDB',
	'1=1'
	);

-- 增量追加
INSERT INTO manager.job_data_sync(
	src_schema_name, src_table_name,
	dst_schema_name, dst_table_name, 
	src_select_statement, 
	sync_mode, 
	src_incr_field, 
	incr_point,
	fields_mapping, 
	inuse,
	src_type, src_db_name,
	src_where_statement 
	)
	VALUES (
	'WIQWMS', 'V_CUS_FPRK_TEMP',
	'wms', 'cus_fprk_temp',
	'FP_NO, ITEM, CARTON_NO, SERIAL_NUMBER, STATUS, INPUT_TIME, UPLOAD_TIME, INPUT_USERID, UPLOAD_USERID, PALLET_NO, LOCATION_ID, PART_ID, IS_SHIPMENT, PRODUCTION_DATE, WH_CODE, PART_NO, CUSTOMER_SN, UPDATE_TIME',
	'append', 
	'UPDATE_TIME', 
	'2022-01-01 00:00:00',
	'{"UPDATE_TIME": "update_time"}', 
	true,
	'CHICONY-WMS', 'MES',
	'${SRC_INCR_FIELD} > to_date(''${INCR_POINT}'', ''yyyy-mm-dd hh24:mi:ss'')'
	);
	
-- 增量合并
INSERT INTO manager.job_data_sync(
	src_schema_name, src_table_name,
	dst_schema_name, dst_table_name, 
	src_select_statement, 
	sync_mode, 
	src_incr_field, 
	dst_pk, 
	incr_point,
	fields_mapping, 
	inuse,
	src_type, src_db_name,
	src_where_statement 
	)
	VALUES (
	'WIQWMS', 'V_CUS_FPRK_TEMP',
	'wms', 'cus_fprk_temp',
	'FP_NO, ITEM, CARTON_NO, SERIAL_NUMBER, STATUS, INPUT_TIME, UPLOAD_TIME, INPUT_USERID, UPLOAD_USERID, PALLET_NO, LOCATION_ID, PART_ID, IS_SHIPMENT, PRODUCTION_DATE, WH_CODE, PART_NO, CUSTOMER_SN, UPDATE_TIME',
	'merge', 
	'UPDATE_TIME', 
	'FP_NO',
	'2022-01-01 00:00:00',
	'{"UPDATE_TIME": "update_time"}', 
	true,
	'CHICONY-WMS', 'MES',
	'${SRC_INCR_FIELD} > to_date(''${INCR_POINT}'', ''yyyy-mm-dd hh24:mi:ss'')'
	);
```

8. 查看数据同步是否正常执行

![](待完善截图)

### 数据库实时同步

随着群电对数据及时性要求的不断提高，同时待同步表也有主键，则会考虑通过 CDC  方式进行数据同步，目前受限于表主键，以及业务场景等因素暂不支持该方式数据同步。

### 共享目录下文件导入

采用 DCAgent + SHELL 方式，适用于外部系统对接，当前 SAP / WF 系统数据文件导入就采用了该方式。

#### 作业流程

#### 前置条件

- 共享目录需挂载到 Linux Server
- 仅支持 **CSV风格** 文件导入，暂不支持 Excel 文件

#### 具体操作

1. 预先挂载 SMB 目录到 Linux Server 目录下

```bash
sudo mkdir /mnt/smb
sudo mount -t cifs //172.25.58.50/mistest /mnt/smb -o uid=1000,username=bdcc,password=Chiconypower123,noatime,_netdev,vers=2.1
ls -ltr /mnt/smb
```

2. 修改 logagent.ini 配置文件，**监听待导入文件所在目录**，以及**自定义数据导入命令**

```ini
hostname = 172.25.57.1

[NORMAL.SAP.SAP-CG-DATA]
watch = /mnt/smb/SAP/SAP-CG-DATA
patterns = WO_
debounce = 3000
command = /opt/bdcc/dcagent/plugins/psql-copy/copy-sap.sh
polling_interval = 30
ignores = Backup

[NORMAL.SAP.Q-Hold]
watch = /mnt/smb/SAP/Q-Hold
debounce = 3000
command = /opt/bdcc/dcagent/plugins/psql-copy/copy-sap-q-hold.sh
polling_interval = 30
ignores = Backup

[NORMAL.WF]
watch = /mnt/smb/WF_defect
debounce = 3000
command = /opt/bdcc/dcagent/plugins/psql-copy/copy-wf.sh
polling_interval = 30
ignores = (Backup|_invalid)
```

```bash
#!/bin/bash
set -e
cd `dirname $0`

datfile=$1

if [ -z "$datfile" ]; then
    echo "Usage: $0 WO_202210280800.txt"
    exit 1
fi

source .env

psql -ac "\copy sap.rework_statistic(plant, wo, model, qty, date1, date2, wo_type, col1, line, release_date, reason, pic, reason_desc) from ${datfile}"
# Rework 数据导入之后，触发整理程序
# psql -ac "select dm.func_scc_rework_report();"

# 导入成功之后，将该文件挪至 Backup 目录
mv $datfile $(dirname $datfile)/Backup/
```

3. 推送 dc-agent 最新部署配置到 Linux Server 

```bash
cd chicony-power-datajobs
./deploy-dcagent.sh
```

4. 登录 Linux Server，启动或重启 dc-agent 服务

```bash
cd /opt/bdcc/dcagent
# 将 dc-agent 注册到系统服务里面，仅在最初部署时执行一次即可
sudo ./dcagent_4.2.1_linux_amd64 install
# 启动
sudo ./dcagent_4.2.1_linux_amd64 start
# or
sudo systemctl start dcagent.service
```

```bash
# 重启
sudo ./dcagent_4.2.1_linux_amd64 restart
# or 
sudo systemctl restart dcagent.service
```

5. 查看 dc-agent 服务运行日志，确认文件导入是否有误

### 消息总线数据导入

考虑到系统解耦，对接 DAP 外部系统采用 Kafka 日志总线，DAP 推送 SPI，NPM等日志消息到 Kafka，下游搭配 Flink 解析任务。除了 DAP，DAVI 网页端文件上传解析导入也采用该方式。

### 第三方应用数据导入

定时同步 微信小程序的云端数据库 是一个定制化任务，此类第三方数据均需要独立开发。

## 数据同步遇到的问题与解决方案

### 增量和全量同步的合并

### 同步性能的处理

