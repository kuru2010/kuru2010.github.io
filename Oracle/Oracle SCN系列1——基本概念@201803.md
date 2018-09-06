---
layout: post
title:  "Oracle SCN系列1——基本概念"
date:   2018-03-24 01:15:15 +0800
categories: Oracle
tags: SCN Oracle
author: 枯荣长老
---

Oracle11gR2 Database Concepts文中关于SCN的解读描述如下：

**System Change Numbers (SCNs)**  
**A system change number (SCN) is a logical, internal time stamp used by Oracle**
**Database.** SCNs order events that occur within the database, which is necessary to
satisfy the ACID properties of a transaction. Oracle Database uses SCNs to mark the
SCN before which all changes are known to be on disk so that recovery avoids
applying unnecessary redo. The database also uses SCNs to mark the point at which
no redo exists for a set of data so that recovery can stop.  
**SCNs occur in a monotonically increasing sequence.** Oracle Database can use an SCN
like a clock because an observed SCN indicates a logical point in time and repeated
observations return equal or greater values. If one event has a lower SCN than another
event, then it occurred at an earlier time with respect to the database. Several events
may share the same SCN, which means that they occurred at the same time with
respect to the database.  
**Every transaction has an SCN.** For example, if a transaction updates a row, then the
database records the SCN at which this update occurred. Other modifications in this
transaction have the same SCN. When a transaction commits, the database records an SCN for this commit.  
**Oracle Database increments SCNs in the system global area (SGA).** When a
transaction modifies data, the database writes a new SCN to the undo data segment
assigned to the transaction. The log writer process then writes the commit record of
the transaction immediately to the online redo log. The commit record has the unique
SCN of the transaction. Oracle Database also uses SCNs as part of its instance
recovery and media recovery mechanisms.   

从这段描述中可以看出：
1. 在Oracle数据库中，SCN是作为内部的逻辑时钟来使用，以保证关系型事务的ACID特性。SCN主要用于并发控制（读一致性，Read Consistency），重做日志的记录顺序，以及数据库的实例恢复与介质恢复。
2. SCN单向顺序增长，每个数据库事务都被赋予一个SCN。事务提交时，数据库会记录事务的SCN。
3. Oracle数据库在SGA中累加SCN。

SCN是数据库一致性状态与事务提交之间关系的标记。每个数据库都有一个全局的SCN生成器。根据Oracle的内部算法，SCN由SCN Base（低位）和SCN Wrap（高位）组成，总共占用六个字节，其中SCN Base占用四个字节，SCN Wrap占用两个字节。每一次变化都会增加SCN Base值，SCN Base达到最大后，会增加SCN Wrap值，然后重置SCN Base值。
SCN= SCN Base + SCN Wrap\*SCN Maximum Reasonable Rate。
在Oracle 11.2.0.2之前，Oracle认为正常情况下，每秒事务提交数不会超过16384（16K，即SCN Maximum Reasonable Rate为16K）。最大的当前SCN限制阈值=（当前时间-1988年1月1日）\*24\*60\*60\*16384。
从Oracle 11.2.0.2开始，SCN Maximum Reasonable Rate被调整为32K。
在Oracle 12c以上版本，SCN改为占用八个字节，以支持更大的当前SCN限制阈值和SCN增速。

**事务与SCN**  
无法通过SCN来识别正在进行的事务。只有事务进行提交时，才会分配SCN。Oracle通过XID（Transaction Identifier，即事务使用的回滚段地址）来识别正在进行的事务。事务开始时，会关联一个快照SCN，以实现读一致性。

**DBLINK与SCN**

当两个数据库通过dblink互联访问时，为了维持分布式事务读一致性需要，Oracle内部会在数据库之间同步SCN，将dblink涉及的库SCN全部调整为这些库中最大的SCN。这就是我们通常所说的SCN传播。如果有某些库曾经采取过特殊手段将SCN递增调整的很大，且处于DBLINK传播链中，就会导致其他库SCN异常增长，极有可能触发经典的错误：ORA-19706: invalid SCN和经典的SCN Headroom现象。在2012年1月的Oracle CPU或者PSU中，增加了_external_scn_rejection_threshold_hours参数，就是为了防止DBLINk引起的SCN不当传播。

**SCN增速（SCN Rate）**：

```
set numwidth 17
set pages 1000
alter session set nls_date_format='DD/Mon/YYYY HH24:MI:SS';

select *
  from (SELECT tim,
               gscn,
               round(rate),
               round((chk16kscn - gscn) / 24 / 3600 / 16 / 1024, 1) "Headroom"
          FROM (select tim,
                       gscn,
                       rate,
                       ((((to_number(to_char(tim, 'YYYY')) - 1988) * 12 * 31 * 24 * 60 * 60) +
                       ((to_number(to_char(tim, 'MM')) - 1) * 31 * 24 * 60 * 60) +
                       (((to_number(to_char(tim, 'DD')) - 1)) * 24 * 60 * 60) +
                       (to_number(to_char(tim, 'HH24')) * 60 * 60) +
                       (to_number(to_char(tim, 'MI')) * 60) +
                       (to_number(to_char(tim, 'SS')))) * (16 * 1024)) chk16kscn
                  from (select FIRST_TIME tim,
                               FIRST_CHANGE# gscn,
                               ((NEXT_CHANGE# - FIRST_CHANGE#) /
                               ((NEXT_TIME - FIRST_TIME) * 24 * 60 * 60)) rate
                          from v$archived_log
                         where (next_time > first_time)))
         order by 3 desc, 1, 2)
 where rownum < 11;
```

