## hive库处理

```
ALTER TABLE `COLUMNS_V2` CHANGE `COMMENT` `COMMENT` VARCHAR(256) CHARACTER SET utf8  DEFAULT NULL;
alter table TABLE_PARAMS modify column PARAM_VALUE mediumtext character set utf8;
```

我这里只改了上面两个字段，如果还有其他字段也可改为utf8

```
alter table CLOUMNS_V2 modify column COMMENT varchar(256) character set utf8; 
alter table TABLE_PARAMS modify column PARAM_VALUE mediumtext character set utf8;
alter table PARTITION_KEYS modify column PKEY_COMMENT varchar(4000) character set utf8; 
alter table PARTITION_PARAMS modify column PARAM_VALUE varchar(4000) character set utf8;
slter table INDEX_PARAMS modify column PARAM_VALUE varchar(4000) character set utf8;
```
这里注意字段类型，不通hive版本字段类型不一致

## 导出sql

```
## 5.1-hive-表
SELECT
    d.OWNER_NAME AS '用户名',
    -- DB_ID,
    d.`NAME` AS '库名',
    -- t.TBL_ID,
    -- t.SD_ID,
    t.TBL_NAME AS '表名',
    t1.`comment` AS '表说明',
    t1.numRows AS '数据量',
    FROM_UNIXTIME(t.CREATE_TIME) AS '创建时间'
FROM
    hive.DBS d JOIN hive.TBLS t USING(DB_ID)
JOIN (SELECT
TBL_ID,
MAX(CASE tp.PARAM_KEY WHEN 'comment' THEN tp.PARAM_VALUE ELSE NULL END) AS `comment`,
MAX(CASE tp.PARAM_KEY WHEN 'numRows' THEN tp.PARAM_VALUE ELSE NULL END) AS `numRows`
FROM hive.TABLE_PARAMS tp
GROUP BY TBL_ID ) t1 USING(TBL_ID)
ORDER BY d.`NAME`,t.TBL_NAME
INTO outfile 'hivetable.txt' FIELDS terminated by '\,';

## 5.2-hive-字段
SELECT
    d.OWNER_NAME AS '用户名',
    -- DB_ID,
    d.`NAME` AS '库名',
    -- t.TBL_ID,
    -- t.SD_ID,
    t.TBL_NAME AS '表名',
    t1.`comment` AS '表说明',
    c1.INTEGER_IDX AS '列ID',
    c1.COLUMN_NAME AS '字段名',
    c1.TYPE_NAME AS '数据类型',
    c1.`COMMENT` AS '字段说明'
FROM
    hive.DBS d JOIN hive.TBLS t USING(DB_ID)
JOIN (SELECT
TBL_ID,
MAX(CASE tp.PARAM_KEY WHEN 'comment' THEN tp.PARAM_VALUE ELSE NULL END) AS `comment`
FROM hive.TABLE_PARAMS tp
GROUP BY TBL_ID ) t1 USING(TBL_ID)
JOIN (SELECT
    s.SD_ID,
    CD_ID,
    c.INTEGER_IDX,
    c.COLUMN_NAME,
    c.TYPE_NAME,
    c.`COMMENT`
FROM
    hive.SDS s
JOIN hive.COLUMNS_V2 c USING (CD_ID)) c1 USING(SD_ID)
ORDER BY d.`NAME`,t.TBL_NAME,c1.INTEGER_IDX
INTO outfile 'hivecolum.txt' FIELDS terminated by '\,';
```

默认存储在mysql的data目录

## 测试

```
create table default.banma_page_cmd 
(  
page_id bigint comment '页面ID',  
page_name string comment '页面名称',  
page_url string comment '页面URL'  
)  
comment '页面视图'  
partitioned by (pt string comment '当前时间，用于分区字段')  
;
```

