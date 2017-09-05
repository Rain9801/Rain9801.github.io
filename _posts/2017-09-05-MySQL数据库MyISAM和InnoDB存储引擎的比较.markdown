---
layout: post
title:  "MySQL数据库MyISAM和InnoDB存储引擎的比较"
date:   2017-09-05 16:15:00 +0800
---
**

MySQL数据库MyISAM和InnoDB存储引擎的比较
----------------------------

**
MyISAM是MySQL的默认存储引擎，基于传统的ISAM类型，支持全文搜索，但不是事务安全的，而且不支持外键。每张MyISAM表存放在三个文件中：frm 文件存放表格定义；数据文件是MYD (MYData)；索引文件是MYI (MYIndex)。

InnoDB是事务型引擎，支持回滚、崩溃恢复能力、多版本并发控制、ACID事务，支持行级锁定（InnoDB表的行锁不是绝对的，如果在执行一个SQL语句时MySQL不能确定要扫描的范围，InnoDB表同样会锁全表，如like操作时的SQL语句），以及提供与Oracle类型一致的不加锁读取方式。InnoDB存储它的表和索引在一个表空间中，表空间可以包含数个文件。

主要区别：

 1. MyISAM是非事务安全型的，而InnoDB是事务安全型的。
 2. MyISAM锁的粒度是表级，而InnoDB支持行级锁定。
 3. MyISAM支持全文类型索引，而InnoDB不支持全文索引。
 4. MyISAM相对简单，所以在效率上要优于InnoDB，小型应用可以考虑使用MyISAM。
 5. MyISAM表是保存成文件的形式，在跨平台的数据转移中使用MyISAM存储会省去不少的麻烦。
 6. InnoDB表比MyISAM表更安全，可以在保证数据不会丢失的情况下，切换非事务表到事务表（alter table tablename type=innodb）。

应用场景：

MyISAM管理非事务表。它提供高速存储和检索，以及全文搜索能力。如果应用中需要执行大量的SELECT查询，那么MyISAM是更好的选择。

InnoDB用于事务处理应用程序，具有众多特性，包括ACID事务支持。如果应用中需要执行大量的INSERT或UPDATE操作，则应该使用InnoDB，这样可以提高多用户并发操作的性能。
