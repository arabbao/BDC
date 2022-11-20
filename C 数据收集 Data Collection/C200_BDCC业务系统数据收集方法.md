<img src="../封面前言和封底%20Cover%20Preface/resource/BDC.png" title="NAME" height="20%" width="20%">

# Big Data Cloud Center Training BDCC培训课程
## 当前课程：C200 BDCC业务系统数据收集方法
### 课程版本： V1.0
### 讲师：张兴龙

# 前言 Preface

## 课程受众

- 数据工程师
- DBA

## 课程目标

- 了解数据库同步怎么新增一张表
- 了解怎么导入一个文本文件

## 课程在BDCC架构中的映射

![](../封面前言和封底%20Cover%20Preface/resource/BDCC-traning-DC1.png)

# 课程内容

## 数据收集方法

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

- 共享目录需预先挂载到 Server 的文件目录
- 仅支持 **CSV风格** 文件导入，暂不支持 Excel 文件

#### 具体操作

1. 预先挂载共享目录到 Server 某个文件目录下，并确定待导入文件的文件路径

```bash
sudo mkdir /mnt/smb
sudo mount -t cifs //172.25.58.50/mistest /mnt/smb -o uid=1000,username=bdcc,password=Chiconypower123,noatime,_netdev,vers=2.1
ls -ltr /mnt/smb
```

2. 本地先修改 logagent.ini 配置文件，配置**监听(watch)的文件目录**，以及自定义**新增文件之后的触发命令(command)**

```ini
hostname = 172.25.57.1

[NORMAL.SAP.STO-CPKC]
watch = /mnt/smb/SAP/STO-CPKC
command = /opt/bdcc/dcagent/plugins/psql-copy/copy-sap-fg-inventory.sh
polling_interval = 30
ignores = (Backup|_invalid)

; ...
```

```bash
#!/bin/bash
# set -e表示一旦脚本中有命令的返回值为非0，则脚本立即退出，后续命令不再执行; 所以此处需要注释掉下面一行
# set -e
cd `dirname $0`

datfile=$1
target_schema="sap"
target_table="fg_inventory"

if [ -z "$datfile" ]; then
    echo "Usage: $0 YRCOA169120221024141608.txt"
    exit 1
fi

source .env

psql -ac "drop table if exists sync.${target_table}; create table sync.${target_table} (like ${target_schema}.${target_table});"
# NOTICE: 若文件中字段顺序和数据库表中字段顺序一致，则可以不用显示地申明字段，否则此处需要改写
psql -ac "\copy sync.${target_table} from ${datfile}"

if [ $? -ne 0 ]; then
    mv $datfile ${datfile}_invalid
    echo "Invalid File: ${datfile}" && exit 3
fi

# NOTICE: \copy 操作成功，才替换原表，如果直接 truncate 可能会造成原表数据被清空
psql -ac "truncate table ${target_schema}.${target_table}; insert into ${target_schema}.${target_table} select * from sync.${target_table};"

# 导入成功之后，将该文件挪至 Backup 目录
mv $datfile $(dirname $datfile)/Backup/
```

3. 推送 dc-agent 最新部署配置到 Server 

```bash
cd chicony-power-datajobs
./deploy-dcagent.sh
```

4. 登录 Server，启动或重启 dc-agent 服务

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

### 共享目录数据文件重复导入

首先确认一下是否是真的重复导入，排查一下源头生成

### 共享目录数据文件没有导入

首先看共享目录下的文件，是否维持原有写入路径，没有转移到 Backup 目录。

## 附录:

### 同步任务配置表

| field_name | description | sample |
| ------ | ------ | ------ |
| job_id | 同步配置自增ID | 1 |
| src_type | 源Server名称 | TAO |
| src_db_name | 源数据库名称 | PCA |
| src_schema_name | 源schema名称 | dbo |
| src_table_name | 源表名 | PCA_SNO |
| dst_schema_name | 目标schema名称 | fis |
| dst_table_name | 目标表名称 | pca_pca_log |
| src_select_statement | 同步字段（逗号隔开，别名方式可做源字段到目标字段的映射） | SnoId,McbSno,WkNo,Model,SWC,PdLine,WC,Rev,PCB,IsPass,NWC,Status,PenNo,Cdt,Udt,MIS_ID |
| src_where_statement | 同步默认条件（默认值为1=1） | 1=1 |
| sync_mode | 同步方式（merge/append/full） | merge |
| src_incr_field | 源表用于增量的指针栏位名称(全量同步时须填写为'1') | Udt |
| dst_pk | 目标表更新依据栏位（merge时生效） | snoid |
| dst_distributed_by | 目标表分布键（选填，仅作记录） | mcbsno |
| fields_mapping | 增量同步指针栏位源字段与目标字段映射 | {"Udt": "udt"} |
| incr_point | 增量指针值（字符形式，可支持数字增量。全量同步时需填写为'-1'）| 2022-06-20 09:00:07.99 |
| cdt | 创建时间 | 2022-06-20 09:33:44.921628 |
| udt | 更新时间（上次成功同步时间）| 2022-06-21 09:33:44.921628 |
| remark | 备注信息 | |
| inuse | 是否开启同步 | true |

# 课后问题

1. 数据库同步新增一张表时，需要明确什么？
2. 若看板数据未更新，如何排查？
3. CSV等文本文件的导入大家更倾向选择哪种方式？

# 讲师邮件地址

Zhang.Xing-Long@inventec.com




