# 数据同步

[TOC]

## 概述

## 应用

### 数据库同步方式系统对接 -- Infra Monitor 任务管理

#### 前置条件

- 若需增量同步，需明确表中是否有一个字段可以作为数据更新的指针，比如 UDT，ID，当更新该表中某一字段的时候，UDT 都会同步修改
- 若需增量合并，需明确表的业务主键是什么？以此作为数据合并的依据。
- 如果初始化同步的数据量较大，超过5kw，最好减少同步的数据量，增加必要的约束条件，避免不必要的 Oracle 读取超时问题。

#### 实践案例

对接 MES / WMS 系统数据库采用了 DB 数据同步方式，实现以表为粒度进行同步。

1. 拷贝模板任务，修改 config.properties 文件

```java
// 任务唯一标识
SRC_TYPE=<JOB_ID>

// 源端为 Oracle 时
SOURCE_DB_URL=jdbc:oracle:thin:@//<HOST>:<PORT>
SOURCE_DB_CLASSNAME=oracle.jdbc.driver.OracleDriver
SOURCE_DB_USER=<USERNAME>
SOURCE_DB_PASSWORD=<PASSWORD>

// 目标端为 PostgreSQL/Greenplum 时
TARGET_DB_URL=jdbc:postgresql://<HOST>:<PORT>/<DATABASE>
TARGET_DB_CLASSNAME=org.postgresql.Driver
TARGET_DB_USER=<USERNAME>
TARGET_DB_PASSWORD=<PASSWORD>
```

2. 明确待同步的表，字段，以及表同步规则

	主要弄明白以下三个问题:

	- 源表同步方式:  要同步哪张表，哪些字段，是选择 `全量同步`/`增量追加`/`增量合并`
	- 增量同步字段:  如果同步方式选择`增量追加`/`增量合并`，需要确定一个自增的字段
	- 主键/业务主键: 如果同步方式选择了`增量合并`，就需要定义一个或多个字段来做数据更新

3. 访问目标端数据库，编写并执行如下SQL

```sql
-- 全量同步
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

-- 增量追加
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
	'${SRC_INCR_FIELD} > to_date(''${INCR_POINT}'', ''yyyy-mm-dd hh24:mi:ss'')'
	);
```

4. 新增定时任务

### 数据库实时同步 -- CDC

#### 前置条件

- 表必须包含主键

#### 实践案例

随着群电对数据及时性要求的不断提高，同时数据表也支持CDC，则会考虑验证该方案，目前暂不考虑。

### 共享目录方式系统对接 -- DCAgent + SHELL

适用于外部系统对接，当前对接 SAP / WF 系统数据文件采用了文件监听加导入的方式。

1. 挂载 smb 目录到 linux 目录下
2. 部署 dcagent ，修改 logagent.ini 配置文件，监听相应的目录
3. 查看 log ，验证文件处理情况

参见: [DCAgent](http://bdcc-infra-grafana.sz.chiconypower.com.cn/d/000000002/dcagent)

### 消息队列方式系统对接 -- Kafka

对接 DAP 外部系统走了 Kafka 日志总线，承接比如 SPI 的日志上传。DAVI 网页端上传文件也依赖 Kafka

### 第三方接口方式系统对接 -- 自定义开发

对接 微信小程序的云端数据库 采用了该方式，另外 SAP 部分数据的导入也可能采用该方式
