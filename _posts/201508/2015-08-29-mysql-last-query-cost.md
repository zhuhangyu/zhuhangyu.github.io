---
layout: post
title:  "记一次网站调优过程"
date:   2015-08-29 23:00:00
categories: mysql
tags: mysql explain Last_query_cost php php-fpm errorlog slowlog
author: "zhuhangyu"
---

Last_query_cost的优化

php errorlog 有大量的如下信息，我这里只截取了一条

[29-Aug-2015 11:48:52] WARNING: [pool www] seems busy (you may need to increase pm.start_servers, or pm.min/max_spare_servers), spawning 16 children, there are 0 idle, and 85 total children
[29-Aug-2015 11:03:08] WARNING: [pool www] child 1155, script '/Data/website/tsy-web/web/index.php' (request: "GET /index.php") execution timed out (13.761323 sec), terminating


看到这里，千万不要以为增大php-fpm的进程数就可以解决问题，要找到根本原因才行。
还有如下信息

[29-Aug-2015 11:03:08] NOTICE: child 1347 stopped for tracing
[29-Aug-2015 11:03:08] NOTICE: about to trace 1347
[29-Aug-2015 11:03:08] NOTICE: finished trace of 1347
[29-Aug-2015 11:03:08] WARNING: [pool www] child 1345 exited on signal 15 (SIGTERM) after 16.655853 seconds from start
[29-Aug-2015 11:03:08] NOTICE: [pool www] child 1485 started


vmstat也看到，free那一列的内存数值很快的变大变小

我理解为：php-fpm收到请求，但是执行很慢，很快达到pm设置的上限。随后不停的有请求进来，php-fpm不得不强行终止超时的请求，终止掉自身进程，重新开启新进程

php-fpm slowlog有大量的如下信息，我这里只截取了一条

[29-Aug-2015 20:35:39]  [pool www] pid 30302
script_filename = /Data/website/tsy-web/web/index.php
[0x00007f68d8f3e608] execute() /Data/website/tsy-web/vendor/yiisoft/yii2/db/Command.php:825



从这里看出，第一条栈的信息，就是具体执行慢的原因，大概和db有关。

同时，从mysql数据库监控看到，cpu使用率很高。

看慢查询日志，有这么一条，被记录了很多次，因为我就压力测试一个页面，居然有的执行花费了54秒

mysql> select * from t_trades where (((gameid=1) and (states=2) and (isdel=3)) and (goodsid=4)) and (count-soldcount>5) order by id desc;
Empty set (0.07 sec)

mysql> show status like '%last_query%';
+--------------------------+--------------+
| Variable_name            | Value        |
+--------------------------+--------------+
| Last_query_cost          | 11169.599000 |
| Last_query_partial_plans | 1            |
+--------------------------+--------------+
2 rows in set (0.00 sec)

返回空结果集，但是0.07s，不是很慢吧，就是Last_query_cost很高，换一下值，找一个返回有结果集的

mysql> select count(*) from t_trades where ( ( ( `gameid` =297 ) and ( `states` =2) and ( `isdel` =0 ) ) and ( `goodsid` = 1) ) and ( count-soldcount > 0 ) order by `id` desc;        
+----------+
| count(*) |
+----------+
|      314 |
+----------+
1 row in set (0.02 sec)

mysql> show status like '%last_query%';
+--------------------------+-------------+
| Variable_name            | Value       |
+--------------------------+-------------+
| Last_query_cost          | 8612.399000 |
| Last_query_partial_plans | 1           |
+--------------------------+-------------+
2 rows in set (0.00 sec)

看着也不慢啊，才0.02s，同样Last_query_cost挺高的，这是为什么呢，explain看一下

mysql> explain select count(*) from t_trades where (((gameid=1) and (states=2) and (isdel=3)) and (goodsid=4)) and (count-soldcount>5) order by id desc;
+----+-------------+----------+------+-----------------------------+-------+---------+-------+------+-------------+
| id | select_type | table    | type | possible_keys               | key   | key_len | ref   | rows | Extra       |
+----+-------------+----------+------+-----------------------------+-------+---------+-------+------+-------------+
|  1 | SIMPLE      | t_trades | ref  | gameid,states,isdel,goodsid | isdel | 2       | const |    1 | Using where |
+----+-------------+----------+------+-----------------------------+-------+---------+-------+------+-------------+
1 row in set (0.01 sec)

居然给四个字段单独做了索引，show indexes from t_trades 看了一下，这四个字段的Cardinality都特别低，都是100以内。

于是删掉这四个索引，建了一个联合索引，再看看explain

mysql> explain select * from `t_trades` where `gameid`=297 order by `discount` desc limit 10; 
+----+-------------+----------+------+---------------------+---------------------+---------+-------+-------+-----------------------------+
| id | select_type | table    | type | possible_keys       | key                 | key_len | ref   | rows  | Extra                       |
+----+-------------+----------+------+---------------------+---------------------+---------+-------+-------+-----------------------------+
|  1 | SIMPLE      | t_trades | ref  | gid_sta_del_goodsid | gid_sta_del_goodsid | 5       | const | 14268 | Using where; Using filesort |
+----+-------------+----------+------+---------------------+---------------------+---------+-------+-------+-----------------------------+
1 row in set (0.00 sec)

嗯，正确使用索引了


mysql> select count(*) from t_trades where ( ( ( `gameid` =297 ) and ( `states` =2) and ( `isdel` =0 ) ) and ( `goodsid` = 1) ) and ( count-soldcount > 0 ) order by `id` desc;
+----------+
| count(*) |
+----------+
|      314 |
+----------+
1 row in set (0.00 sec)

时间在10ms之下了，不错哦，再看看Last_query_cost

mysql> show status like '%last_query%';
+--------------------------+------------+
| Variable_name            | Value      |
+--------------------------+------------+
| Last_query_cost          | 385.199000 |
| Last_query_partial_plans | 1          |
+--------------------------+------------+
2 rows in set (0.00 sec)

减少了1个数量级哦



再重新进行压力测试，这回效果非常好，没有任何问题了