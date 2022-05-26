# 系统数据库 mysql
#MySQL 

系统数据库 mysql 中包含 MySQL 服务器运行时所需的信息。其中，我们常用的系统表

## 授权系统表

这些系统表中包含有关用户账户及其特权的授予信息。

+ user：用户账号、全局特权和其他非特权的列。
+ db：数据库特权。
+ tables_priv：表级特权。
+ columns_priv：列级特权。

## 日志系统表

服务器使用以下系统表进行记录：

+ general_log：通用查询日志表。
+ slow_log：慢查询日志。

> 日志表使用 CSV 存储引擎。

## 时区系统表

这些系统表包含时区信息：

+ time_zone：时区 ID 以及是否使用闰秒。
+ time_zone_name：时区 ID 和名称之间的映射。
+ time_zone_transition，time_zone_transition_type：时区描述。

## 优化器系统表

这些系统表供优化器使用：

+ innodb_index_stats，innodb_table_stats：用于 InnoDB 持久优化器统计。
+ server_cost，engine_cost：[[慢查询优化 - 基于代价的优化器|基于代价的优化器]]使用的基本代价参数值。

