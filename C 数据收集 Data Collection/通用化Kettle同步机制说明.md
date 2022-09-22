### 《Kettle批量同步方案——说明文档》
#### 方案介绍
该方法以Kettle作为基础开发，实现了一套通用化的批量数据同步Job。Job通过访问配置信息，可以对两个数据库实例之间的多张表进行全量、增量同步。  
目前支持SQL Server、PG、GP之间的相互同步，支持merge（增量更新）、append（增量追加）、full（全量）同步方式。
#### 方案设计思路
![6216451a1efad407c7ca3553](http://assets.processon.com/chart_image/6216451a1efad407c7ca3553.png?_=1655794345440)
> 注：考虑到全量同步的表绝大多数均为小表无索引，full情况下实际才用了truncate+insert的方式。根据实际情况选择。
#### Kettle Job组成
根据以上设计实现的Kettle Job文件由2个Job文件、3个Transform文件组成：  
![image](/uploads/2e88e0667ddb6e150b22f6a5005152f0/image.png)  

- **{Target_Server}_job_data_sync.kjb**  
![image](/uploads/18c1fbf4e832543347c567b183351ba3/image.png)  
总任务入口，调用**get_job_list.ktr**获取同步表的配置信息列表，并按照并行度设定依次启动每张表的同步子任务**data_sync_by_table.kjb**。  
> 注：由于Kettle并行度的设定必须在ktr中，所以并行调度子任务部分放在了**get_job_list.ktr**中。  

- **get_job_list.ktr**  
![image](/uploads/82f7e0a9bb7a609df7836ab331adbcb6/image.png)  
访问目标端上的配置表获取该类任务的同步表配置信息List，并按照并行度设定依次启动每张表的同步子任务**data_sync_by_table.kjb**。  

- **data_sync_by_table.kjb**  
![image](/uploads/f70134b3d2796c0ec81fb92cd4260fe3/image.png)  
负责完成每张表的具体同步动作。

- **set_var**  
![image](/uploads/79a54b53a243adce0ed5fd64edabe8b9/image.png)  
执行**set_var.ktr**文件，将同步表的配置信息存储为Kettle变量，供后续使用。  

- **recreate_mid_table**  
![image](/uploads/0020859728be398ee286ab5d3a95f6c8/image.png)  
执行sql脚本重建同步中间表。  

- **sync_mid_table**  
![image](/uploads/b290912feb72c9f153fdbac48c42211d/image.png)  
执行**sync_mid_table.ktr**文件。从源头获取需同步的数据，写入中间表。同时，根据是否有新数据产生，修改新数据标识变量isnull的值（True表示无数据，False表示有新增数据）。  

- **Simple evaluation**  
根据isnull变量值，判断是否有新数据产生。有数据则执行**sync_data**，无数据则子任务结束。  

- **sync_data**  
![image](/uploads/4bff3b8f9729aa0c7211e21fc227aea8/image.png)   
根据sync_mode将新数据由中间表同步至目标表，并更新同步指针。  

#### 同步任务配置表
```sql
CREATE TABLE manager.job_data_sync
(
    job_id serial NOT NULL,
    src_type character varying(50) NOT NULL,
    src_db_name character varying(50) NOT NULL,
    src_schema_name character varying(100) NOT NULL,
    src_table_name character varying(100) NOT NULL,
    dst_schema_name character varying(100 NOT NULL,
    dst_table_name character varying(100) NOT NULL,
    src_select_statement text NOT NULL DEFAULT '*'::text,
    src_where_statement text NOT NULL DEFAULT '1=1'::text,
    sync_mode character varying(10) NOT NULL,
    src_incr_field character varying(100),
    dst_pk character varying(100),
    dst_distributed_by character varying(100) NOT NULL,
    fields_mapping jsonb,
    incr_point text,
    cdt timestamp without time zone NOT NULL DEFAULT now(),
    udt timestamp without time zone NOT NULL DEFAULT now(),
    remark text,
    inuse boolean NOT NULL DEFAULT true,
    CONSTRAINT job_data_sync_pkey PRIMARY KEY (src_type, dst_schema_name, dst_table_name)
)
```  
- 各field含义及示例：  

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
  
#### 使用说明
1. 需提前在目标Server段建立sync schema用于存储**同步中间表**使用；
2. 配置表建立在目标端，且同一目标端可共用一张配置表；
3. 当前版本，每个不同的源Server到目标Server的同步，均需要部署新的同步Job，并修改相应设定。包括：（~~后续会考虑升级，做到共用~~ 该部分新版本已实现）
  - 修改get_job_list中的获取同步List的SQL中where条件src_type = '{源Server名称}'；
  - 修改job中所有的数据源连接——源端连接和目标端连接；