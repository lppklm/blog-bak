title: Mysql XA分布式事务(外部XA)
author: John Doe
tags:
  - Mysql
categories:
  - Mysql
  - ''
date: 2019-01-13 10:01:00
---
# XA 跨数据库事务
>***先声明两个概念：***

**资源管理器（resource manager）**：用来管理系统资源，是通向事务资源的途径。数据库就是一种资源管理器。资源管理还应该具有管理事务提交或回滚的能力。
**事务管理器（transaction manager）**：事务管理器是分布式事务的核心管理者。事务管理器与每个资源管理器（resource manager）进行通信，协调并完成事务的处理。事务的各个分支由唯一命名进行标识。

mysql在执行分布式事务（外部XA）的时候，mysql服务器相当于xa事务资源管理器，与mysql链接的客户端相当于事务管理器。

>***分布式事务原理：分段式提交***

**准备阶段：** 即所有的参与者准备执行事务并锁住需要的资源。参与者ready时，向transaction manager报告已准备就绪
**提交阶段：** 当transaction manager确认所有参与者都ready后，向所有参与者发送commit命令
![图示](https://img-blog.csdn.net/20170425103341298?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc29vbmZseQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast "分段提交")

>***Mysql的XA事务分为外部XA和内部XA***

**外部XA:** 用于跨多MySQL实例的分布式事务，需要应用层作为协调者，通俗的说就是比如我们在PHP中写代码，那么PHP书写的逻辑就是协调者。应用层负责决定提交还是回滚，崩溃时的悬挂事务。MySQL数据库外部XA可以用在分布式数据库代理层，实现对MySQL数据库的分布式事务支持，例如开源的代理工具：网易的DDB，淘宝的TDDL等等。
**内部XA:** 事务用于同一实例下跨多引擎事务，由Binlog作为协调者，比如在一个存储引擎提交时，需要将提交信息写入二进制日志，这就是一个分布式内部XA事务，只不过二进制日志的参与者是MySQL本身。Binlog作为内部XA的协调者，在binlog中出现的内部xid，在crash recover时，由binlog负责提交。(这是因为，binlog不进行prepare，只进行commit，因此在binlog中出现的内部xid，一定能够保证其在底层各存储引擎中已经完成prepare)

>***MySQL XA事务基本语法***

XA {START|BEGIN} xid [JOIN|RESUME] 启动xid事务 (xid 必须是一个唯一值; 不支持[JOIN|RESUME]子句) 
XA END xid [SUSPEND [FOR MIGRATE]] 结束xid事务 ( 不支持[SUSPEND [FOR MIGRATE]] 子句) 
XA PREPARE xid 准备、预提交xid事务 
XA COMMIT xid [ONE PHASE] 提交xid事务 
XA ROLLBACK xid 回滚xid事务 
XA RECOVER 查看处于PREPARE 阶段的所有事务

>***PHP调用MYSQL XA事务示例***
---
**首先要确保mysql开启XA事务支持**
```
SHOW VARIABLES LIKE '%xa%'
SET innodb_support_xa = ON
```
**代码**
```
<?PHP
$dbtest1 = new mysqli("172.20.101.17","public","public","dbtest1")or die("dbtest1 连接失败");
$dbtest2     = new mysqli("172.20.101.18","public","public","dbtest2")or die("dbtest2 连接失败");

//为XA事务指定一个id，xid 必须是一个唯一值。
$xid = uniqid("");

//两个库指定同一个事务id，表明这两个库的操作处于同一事务中
$dbtest1->query("XA START '$xid'");//准备事务1
$dbtest2->query("XA START '$xid'");//准备事务2

try {
    //$dbtest1
    $return = $dbtest1->query("UPDATE member SET name='唐大麦' WHERE id=1") ;
    if($return == false) {
       throw new Exception("库dbtest1@172.20.101.17执行update member操作失败！");
    }

    //$dbtest2
    $return = $dbtest2->query("UPDATE memberpoints SET point=point+10 WHERE memberid=1") ;
    if($return == false) {
       throw new Exception("库dbtest1@172.20.101.18执行update memberpoints操作失败！");
    }

    //阶段1：$dbtest1提交准备就绪
    $dbtest1->query("XA END '$xid'");
    $dbtest1->query("XA PREPARE '$xid'");
    //阶段1：$dbtest2提交准备就绪
    $dbtest2->query("XA END '$xid'");
    $dbtest2->query("XA PREPARE '$xid'");

    //阶段2：提交两个库
    $dbtest1->query("XA COMMIT '$xid'");
    $dbtest2->query("XA COMMIT '$xid'");
} 
catch (Exception $e) {
    //阶段2：回滚
    $dbtest1->query("XA ROLLBACK '$xid'");
    $dbtest2->query("XA ROLLBACK '$xid'");
    die($e->getMessage());
}

$dbtest1->close();
$dbtest2->close();

?>
```

>***XA的性能问题***
>
XA的性能很低。一个数据库的事务和多个数据库间的XA事务性能对比可发现，性能差10倍左右。因此要尽量避免XA事务，例如可以将数据写入本地，用高性能的消息系统分发数据。或使用数据库复制等技术。只有在这些都无法实现，且性能不是瓶颈时才应该使用XA。

>参考:https://blog.csdn.net/soonfly/article/details/70677138