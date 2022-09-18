# 数据同步

[TOC]

## 概述

有哪些方式，每种方式是什么样的一个形式，目前主要有哪几种。

## 应用

### 数据库定时同步 -- Kettle

原理介绍，然后举个示例，申明一下该方法局限性

#### 实现原理

#### 前置条件

- 若需增量同步，需明确表中是否有一个字段可以作为数据更新的指针，比如 UDT，ID，当更新该表中某一字段的时候，UDT 都会同步修改
- 若需增量合并，需明确表的业务主键是什么？以此作为数据合并的依据。
- 如果初始化同步的数据量较大，超过5kw，最好减少同步的数据量，增加必要的约束条件，避免不必要的 Oracle 读取超时问题。

#### 示例

- 拷贝模板任务，修改 config.properties 文件（可选）

> 如果当前启用的任务不能覆盖新增数据源（源端DB），则需要新建一个任务

```java
SRC_TYPE=<CUSTOME_SRC_TYPE>

SOURCE_DB_URL=jdbc:oracle:thin:@//<IP>:1521
SOURCE_DB_CLASSNAME=oracle.jdbc.driver.OracleDriver
SOURCE_DB_USER=<USERNAME>
SOURCE_DB_PASSWORD=<PASSWORD>

TARGET_DB_URL=jdbc:postgresql://172.25.57.1:5493/chicony
TARGET_DB_CLASSNAME=org.postgresql.Driver
TARGET_DB_USER=bdcuser
TARGET_DB_PASSWORD=bdcc+chicony
```

- 明确要同步哪些表，以及相应的同步规则

核心就是明确几个概念，同步方式是选择"全量同步"，"增量追加"，"增量合并"; 源端表有哪些字段需要同步; 源端表的增量指针可以用哪个字段标示。

```sql
INSERT INTO manager.job_data_sync(
	src_schema_name, src_table_name,
	dst_schema_name, dst_table_name, 
	src_select_statement, 
	sync_mode, 
	src_incr_field, 
	dst_pk, dst_distributed_by, 
	incr_point,
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
	'', 
	'', '',
	'',
	'{}', 
	true,
	'CHICONY', 'MESDB',
	'1=1'
	);

INSERT INTO manager.job_data_sync(
	src_schema_name, src_table_name,
	dst_schema_name, dst_table_name, 
	src_select_statement, 
	sync_mode, 
	src_incr_field, 
	dst_pk, dst_distributed_by, 
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
	'', '',
	'2022-01-01 00:00:00',
	'{"UPDATE_TIME": "update_time"}', 
	true,
	'CHICONY-WMS', 'MES',
--	'${SRC_INCR_FIELD} > to_timestamp(''${INCR_POINT}'', ''yyyy-mm-dd hh24:mi:ss.ff3'')',
	'${SRC_INCR_FIELD} > to_date(''${INCR_POINT}'', ''yyyy-mm-dd hh24:mi:ss'')'
	);
```

- Infra Monitor 新增定时任务(可选) 

> 如果当前启用的任务不能覆盖新增数据源（源端DB），则需要新建一个任务

- Infra Monitor 任务测试

### 数据库实时同步 -- CDC

因诸多限制，尚未在群电落地

#### 前置条件

- 表必须包含主键，

### 共享目录文件导入 -- DCAgent + SHELL

适用于外部系统对接，采用文件方式进行解耦。。。

- 挂载 smb 目录到 linux 目录下
- 部署 dcagent ，修改 logagent.ini 配置文件，监听相应的目录
- 查看 log ，验证文件处理情况

### 产测文件自动上传 API -- Kafka

DAP

### Web 文件手动上传 API -- DAVI

面向用户整理的数据

### 第三方开放接口 -- 接口

微信或SAP
