---
title: MySql where与where in
date: 2019-03-14 10:58:06
tags: MySql
categories: 技术
---

### 简介

前两天在写一个业务的时候，会先从A表取出一堆ID，再拿这堆ID去B表里取详细数据，实现这个业务有两种选择。  
1. 遍历ID列表，逐一去B表取。(如果ID数量过多，则会执行很多次数据库查询)  
2. 直接使用MySql where in语句。(只需要查询一次，而且需要保证查出来的数据的顺序与ID列表的顺序一样)  

所以我就很纠结这两种实现的性能是什么样的，哪个最优?  
<!-- more -->
实践出真理，环境如下(框架全是默认设置，无优化):  
> Windows、MySql5.7、laravel-illuminate/database、  

### 数据填充

#### 表结构

```mysql
CREATE TABLE `t_company` (
  `company_id` varbinary(64) NOT NULL COMMENT '公司id',
  `type` tinyint(3) unsigned NOT NULL COMMENT '类型，1：人建的公司、2：爬取的公司',
  `qxb_id` varbinary(64) NOT NULL COMMENT '启信宝id',
  `name` varchar(128) NOT NULL COMMENT '公司名称',
  `short_name` varchar(64) NOT NULL COMMENT '公司简称',
  `logo` varbinary(64) NOT NULL COMMENT '公司Logo',
  `industry_code_list` varbinary(64) NOT NULL COMMENT '所属行业',
  `city_code` varbinary(16) NOT NULL COMMENT '所在城市',
  `address` varchar(128) NOT NULL COMMENT '所在地址',
  `financing` tinyint(3) unsigned NOT NULL COMMENT '融资阶段',
  `scale` tinyint(3) unsigned NOT NULL COMMENT '公司规模',
  `description` varchar(1024) NOT NULL COMMENT '公司简介',
  `create_time` bigint(20) unsigned NOT NULL,
  `update_time` bigint(20) unsigned NOT NULL,
  PRIMARY KEY (`company_id`),
  UNIQUE KEY `u_qxb_id_type` (`qxb_id`,`type`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='公司';
```

#### 数据5W条

```php
    $i = 50000;
    while ($i--) {
        $db->insert([
            'company_id' => random(32),
            'type' => rand(1, 2),
            'qxb_id' => random(),
            'name' => '公司' . random(8),
            'short_name' => 'short',
            'logo' => random(),
            'industry_code_list' => json_encode(['xxx001', 'xxx002']),
            'city_code' => rand(3000, 4000),
            'address' => '地址' . random(5),
            'financing' => rand(1, 7),
            'scale' => rand(1, 7),
            'description' => random(32),
            'create_time' => time(),
            'update_time' => time(),
        ]);
    }
```

### 测试-逐条查询

```php
    $n = 100;
    $id_list = $db->offset(0)->limit($n)->pluck('company_id')->toArray();
    $start_time_ms = time_ms();
    foreach ($id_list as $id) {
        $db->where('company_id', $id)->get();
    }
    $end_time_ms = time_ms();
    echo $end_time_ms - $start_time_ms;
```

#### 结果(单位秒)

数据量|第一次|第二次|第三次|第四次|第五次
:-:|:-:|:-:|:-:|:-:|:-:
100|0.137|0.137|0.133|0.134|0.138
500|3.073|3.158|2.914|3.007|2.965
1000|15.385|17.961|15.261|16.347|16.134


### 测试-where in

```php
    $n = 100;
    $id_list = $db->offset(0)->limit($n)->pluck('company_id')->toArray();
    $start_time_ms = time_ms();
    $db->whereIn('company_id', $id_list);
    $end_time_ms = time_ms();
    echo $end_time_ms - $start_time_ms;
```

#### 结果(单位秒)

数据量|第一次|第二次|第三次|第四次|第五次
:-:|:-:|:-:|:-:|:-:|:-:
100|0.04|0.03|0.03|0.02|0.03
500|0.18|0.20|0.20|0.19|0.20
1000|0.52|0.62|0.52|0.52|0.51

### 总结

可以看出，方案二明显比方案一快了不至数倍，因为执行一条SQL会花费一定的时间在建立连接上。  
所以建议大家有这种需求的时候应该使用性能更高的in语句，能节省大量时间。  


*小插曲*
> MySQL如何以与whereIn子句相同的顺序排序结果？
> 答：可以使用order by field。

```php
    $id_list = $db->offset(0)->limit(1000)->pluck('company_id')->toArray();
    $id_list_str = '\'' . implode('\',\'', $id_list) . '\'';
    $db->whereIn('company_id', $id_list)->orderByRaw('FIELD(company_id, '. $id_list_str . ')')->get();
```
