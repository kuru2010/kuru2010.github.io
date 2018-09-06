---
layout: post
title:  "SCN的时空陷阱——Oracle史上规模最大的补丁升级召集令"
date:   2018-03-20 10:15:18 +0800
categories: Oracle
tags: SCN Oracle
author: 枯荣长老
---

SCN的时空陷阱——Oracle史上规模最大的补丁升级召集令

Oracle最近发布了：

 **(Doc ID 2361478.1) Recommended patches and actions for Oracle databases versions 12.1.0.1, 11.2.0.3 and earlier  –  before June 2019**

一石激起千层浪，升级涉及的版本之多、范围之广，对此引起的影响恐怕已超出Oracle的预期。该文档标题也从最初发布时提到的**Mandatory（强制性的）**等，措辞也修正为较缓和的**“建议或强烈建议（Recommended or Strongly recommended)”。Deadline也从之前的Apr 2019调整到June 2019**。

对于SCN补丁涉及的内部细节，网上有很多专家做了更深层次的解读。这里仅就该文档做一下解读。

Purpose

This support note provides additional info related to recommended patching requirement for Oracle databases versions 11.1.0.7, 11.2.0.3 & 12.1.0.1, to be done before June 2019.

**影响的数据库版本：Databases versions 11.1.0.7, 11.2.0.3 & 12.1.0.1，当然还有Oracle 10.2.x。**至于更低版本的Oracle，鉴于Oracle数据库连接互访的兼容性规定，不建议Oralce 10.2以前的版本和Oracle 11.2以上版本之间出现连接互访的情况。

Scope
 The document is intended for all DBAs.

Details

Oracle Database versions 11.1.0.7, 11.2.0.3 & 12.1.0.1 are strongly recommended to be patched to the patchset/PSU levels mentioned below before June 2019 to **resolve potential future issues in terms of interoperability of dblinks.** No action is needed if you are running database releases/versions 12.2, 12.1.0.2 or 11.2.0.4. If you are still using 10.2 or earlier releases and using dblinks with later database releases, this note also applies.

**The interoperability of database clients with database servers is not impacted.**

**说明：** 

​    **主要影响dblink的访问，数据库客户端和服务间之间的连接操作不受此影响。**

​     **Oracle 12.2, 12.1.0.2, 11.2.0.4以上的数据库已修正，无需升级。**

​     **Oracle 11.1.0.7， 11.2.0.3, 12.1.0.1数据库则是强烈建议尽快安装相应补丁。**

​     **Oracle 10.2及以下的数据库，除非有相应补丁，否则不建议与高版本的数据库建立dblink访问。**

(Based on customer feedback, we are currently evaluating the need and feasibility of providing a patch for 10.2.0.5 and this note will be updated with that information at a later time.)

**说明：稍后会单独提供Oracle 10.2.0.5所需补丁。——我们没有遗忘Oracle 10.2.0.5**

1. What are the recommended patchset/PSU/BP/RU levels ?

For database versions 11.1.0.7, 11.2.0.3 & 12.1.0.1, ensure that all interconnected databases are in the below mentioned patchset/ PSU/BP levels or above:

In summary, 12.2.0.1 and higher releases, 11.2.0.4 and 12.1.0.2 patchsets have this fix included, while patches are available for 11.1.0.7 and 11.2.0.3 releases. If you have any other database server installations (e.g. 10.2.0.5, 11.2.0.2), you should be aware about potential dblink issues in future and consider applying the required patches or upgrading the databases, or not using dblinks with newer versions of databases.

1. What is the timeline for moving to the recommended patchset/PSU/BP mentioned ?

All databases are recommended to be at the above-mentioned release/patchset/ PSU/BP levels (or above) before June 2019.

1. What is the change introduced by the patches listed above?

These patches increase the database's current maximum SCN (system change number) limit.

At any point in time, the Oracle Database calculates a "not to exceed" limit for the number of SCNs a database can have used, **based on the number of seconds elapsed since 1988**. This is known as the database's current maximum SCN limit. Doing this ensures that Oracle Databases will ration SCNs over time, allowing over **500 years of data processing** for any Oracle Database.
These recommended patches enable the databases to allow for a higher current maximum SCN limit. **The rate at which this limit is calculated can be referred to as the “SCN rate”** and these patches help allow higher SCN rates to enable databases to support many times higher transaction rates than earlier releases.

**说明：**

​    **补丁主要调整了两项内容：**

1. **当前SCN限制阈值。当前SCN限制阈值是根据1988以来的秒数与16384的乘积来计算所得，如果数据库事务非常多且极为频繁、或者出现bug时，可能会导致SCN在某一时刻逐渐逼近该时刻的SCN限制阈值，造成数据库无法正常工作。**


1. **根据调整后的最大SCN限制，提供更高的SCN增长速率，以支持更多的并发事务数。**

Please note that the patches only increase the max limit but the current SCN is not impacted. So, if all your databases don’t have any major change in transaction rate, the current SCN would still remain below the current maximum SCN limit and database links between newer (or patched) and unpatched databases would continue to work. The patches provide the safety measure to ensure that you don’t have any issue with dblinks independent of any possible future change in your transaction rate.

**说明：**

   **如果并发事务数没有发生显著变化，当前的SCN保持低于当前SCN限制阈值，那么高版本数据库（或者已修正的数据库）和未修正的数据库之间的dblink仍然可以继续使用。修正补丁不会影响当前的SCN使用，修正补丁只是在将来并发事务数出现显著增长时提供了更为安全的措施来确保dblink访问不会出现问题**

With the patches applied, this change in current maximum SCN limit will happen automatically starting 23rd June 2019.

**说明：**

   **补丁修正后，当前SCN最大限制阈值调整自动于2019年6月23日生效。**

1. What happens if the recommended PSU / patchset is not applied?

If this patch is not applied, the unpatched database will have a lower SCN rate or lower current max SCN limit.

**说明：**

   **未修正的数据库仍然保持现有的SCN机制，只是相对新的SCN机制来说，SCN增速和当前SCN最大限制阈值都比较低。**

The newer or patched databases will have higher SCN rate or higher current max SCN limit.
Therefore, there can be situations when the patched database is at a higher SCN level (due to the higher SCN rate allowance) and the unpatched database is at a much lower SCN level (due to lower SCN rate allowance).
When you open a dblink between these two databases, it has to sync the SCN levels of both the databases and if the SCN increase needed in the unpatched database for this sync is beyond it’s allowed SCN rate or current max SCN limit, then the dblink connection cannot be established.

This situation will not rise immediately after the change, but can potentially arise any time after 23rd June 2019.

**说明：**

   **介于新SCN机制的数据库与老SCN机制的数据库之间的dblink访问，根据dblink机制，会在访问时，同步两个数据库的SCN，以保证数据访问的一致性。这样有可能造成老SCN机制的数据库出现SCN headroom问题，从而导致dblink连接访问被拒绝。这种情况在2019年6月23日以后会更为明显。**

1. What about databases that are 10.2 or older, which are not listed in the table?

You should be aware about potential dblink issues in future and consider about upgrading the databases or not using dblinks with newer versions of databases . If you continue to have such database links after June 2019, you may get run-time errors during database link operations (as explained above) and you would need to disconnect those database links at that time.

(Based on customer feedback, we are currently evaluating the need and feasibility of providing a patch for 10.2.0.5 and this note will be updated with that information at a later time.)

1. How can I check the details regarding the dblinks to and from a database?

In order to identify database links, please review "Viewing Information About database Links" in Database Administrator's guide.
Please note that outgoing db links from a database can be identified via DBA_DB_LINKS view for all database releases.
select * from dba_db_links;

For 12.1 and later releases, you can also find out about incoming database links via DBA_DB_LINK_SOURCES view.
select * from dba_db_link_sources;

1. Will there be any issues with the db links connecting two unpatched databases ? Or databases of older versions?

The dblink connections involving two unpatched databases or two older releases are not affected by this change.

**说明：**

   **两个均未修正的数据库或者老版本数据库之间的dblink访问不受此SCN机制调整影响。**

   **例如两个10.2.0.5数据库，均未修复过该补丁，则这两个数据库之间的dblink访问不受影响**

1. Will the dblinks involving a patched and an unpatched database, stop working immediately after June 2019? 

DB Links will not become unusable immediately after June 2019. However, might encounter errors in situations explained in question 4, at any point in time after June 2019.

**说明：**

   **在已修正的数据库和未修正的数据库之间的dblink访问不会在2019年6月之后立刻出现无法使用的情况，具体可参见第4点。**

1. What should I do, if the dblink connection from an older version database to a latest (or patched) version database fails, after June 2019?

Patch or upgrade the older version database to any patch level mentioned in the table.

**说明：**

   **如果2019年6月以后出现dblink连接因SCN问题导致被拒的情况，可将老版本数据库升级或者安装修正补丁（如果有的话）。**

1. What do we need to do for 11.2.0.4, 12.1.0.2 and 12.2.0.1 database releases?

No action is necessary. All the fixes needed are already included in these releases.

1. Support and Questions

 If you have any queries please post them in the Database community page: https://community.oracle.com/message/14710245#14710245



**简单概括：**

1. **此次修正没有那么可怕。**
2. **此次修正只会影响到dblink访问，不会对数据库客户端的访问有任何影响。**
3. **如果不涉及dblink访问，老版本数据库可以不考虑此次修正，不影响正常使用。**
4. **对目前与新版本数据库（新SCN机制）有建立dblink访问的老版本数据库（未修正，老SCN机制），如果两个库的并发事务数比较稳定且不高，即使老版本数据库不修正，一般也不会出现问题。日常可保持SCN监控来跟踪，以决定将来采取的措施。**



参考文档：

System Change Number (SCN), Headroom, Security and Patch Information (Doc ID 1376995.1)




