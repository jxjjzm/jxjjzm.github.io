### MySql数据库之面试篇 ###
***

### 一、MYSQL 索引及SQL优化 ###


- [MySql数据库之索引](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Database%E7%B3%BB%E5%88%97/Mysql/MySql%E6%95%B0%E6%8D%AE%E5%BA%93%E4%B9%8B%E7%B4%A2%E5%BC%95.md)
- [MySql数据库之SQL优化](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Database%E7%B3%BB%E5%88%97/Mysql/MySql%E6%95%B0%E6%8D%AE%E5%BA%93%E4%B9%8BSQL%E4%BC%98%E5%8C%96.md)






### 二、MYSQL事务与锁 ###


- [MySql数据库之事务](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Database%E7%B3%BB%E5%88%97/Mysql/MySql%E6%95%B0%E6%8D%AE%E5%BA%93%E4%B9%8B%E4%BA%8B%E5%8A%A1.md)
- [MySql数据库之锁](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Database%E7%B3%BB%E5%88%97/Mysql/MySql%E6%95%B0%E6%8D%AE%E5%BA%93%E4%B9%8B%E9%94%81.md)




### 三、其他 ###




- [数据库的三范式是什么？什么是反模式？](http://blog.720ui.com/2017/mysql_core_07_anti-pattern/)
- [MySQL 有哪些数据类型？](https://www.runoob.com/mysql/mysql-data-types.html)


面试题：金额(金钱)相关的数据，选择什么数据类型？

- 方式一，使用 int 或者 bigint 类型。如果需要存储到分的维度，需要 *100 进行放大。
- 方式二，使用 decimal 类型，避免精度丢失。如果使用 Java 语言时，需要使用 BigDecimal 进行对应。



面试题：一张表，里面有 ID 自增主键，当 insert 了 17 条记录之后，删除了第 15,16,17 条记录，再把 MySQL 重启，再 insert 一条记录，这条记录的 ID 是 18 还是 15？



- 一般情况下，我们创建的表的类型是 InnoDB ，如果新增一条记录（不重启 MySQL 的情况下），这条记录的 ID 是18 ；但是如果重启 MySQL 的话，这条记录的 ID 是 15 。因为 InnoDB 表只把自增主键的最大 ID 记录到内存中，所以重启数据库或者对表 OPTIMIZE 操作，都会使最大 ID 丢失。


- 但是，如果我们使用表的类型是 MyISAM ，那么这条记录的 ID 就是 18 。因为 MyISAM 表会把自增主键的最大 ID 记录到数据文件里面，重启 MYSQL 后，自增主键的最大 ID 也不会丢失。（最后，还可以跟面试官装个 x ，生产数据，不建议进行物理删除记录。）


- [MySQL存储引擎－－MyISAM与InnoDB区别](https://github.com/muyinchen/woker/blob/master/mysql/MySQL%E5%AD%98%E5%82%A8%E5%BC%95%E6%93%8E%EF%BC%8D%EF%BC%8DMyISAM%E4%B8%8EInnoDB%E5%8C%BA%E5%88%AB.md)


面试题：MySQL 数据库 CPU 飙升到 500% 的话，怎么处理？

当 CPU 飙升到 500% 时，先用操作系统命令 top 命令观察是不是 mysqld 占用导致的，如果不是，找出占用高的进程，并进行相关处理。（同理，如果此时是 IO 压力比较大，可以使用 iostat 命令，定位是哪个进程占用了磁盘 IO 。）

如果是 mysqld 造成的，使用 show processlist 命令，看看里面跑的 Session 情况，是不是有消耗资源的 SQL 在运行。找出消耗高的 SQL ，看看执行计划是否准确， index 是否缺失，或者实在是数据量太大造成。一般来说，肯定要 kill 掉这些线程(同时观察 CPU 使用率是否下降)，等进行相应的调整(比如说加索引、改 SQL 、改内存参数)之后，再重新跑这些 SQL。（也可以查看 MySQL 慢查询日志，看是否有慢 SQL 。）

也有可能是每个 SQL 消耗资源并不多，但是突然之间，有大量的 Session 连进来导致 CPU 飙升，这种情况就需要跟应用一起来分析为何连接数会激增，再做出相应的调整，比如说限制连接数等。


附：

- [MySQL常见问题总结](https://blog.csdn.net/DERRANTCM/article/details/51534411)




















































































