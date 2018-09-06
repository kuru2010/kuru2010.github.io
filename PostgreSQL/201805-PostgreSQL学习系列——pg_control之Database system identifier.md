---
layout: post
title:  "PostgreSQL学习系列——pg_control之Database system identifier"
date:   2018-05-25 01:00:00 +0800
categories: PostgreSQL
tags: pg_control
author: 枯荣长老
---



```
[hgps@ps2 bin]$ pg_controldata ../data
pg_control version number:            942
Catalog version number:               201510051
Database system identifier:           6548580191788017147
Database cluster state:               in production
pg_control last modified:             Mon 21 May 2018 11:56:42 AM CST
Latest checkpoint location:           0/2797010
Prior checkpoint location:            0/17FAF50
Latest checkpoint's REDO location:    0/2797010
Latest checkpoint's REDO WAL file:    000000010000000000000002
Latest checkpoint's TimeLineID:       1
Latest checkpoint's PrevTimeLineID:   1
Latest checkpoint's full_page_writes: on
Latest checkpoint's NextXID:          0/1836
Latest checkpoint's NextOID:          24576
Latest checkpoint's NextMultiXactId:  1
Latest checkpoint's NextMultiOffset:  0
Latest checkpoint's oldestXID:        1825
Latest checkpoint's oldestXID's DB:   1
Latest checkpoint's oldestActiveXID:  0
Latest checkpoint's oldestMultiXid:   1
Latest checkpoint's oldestMulti's DB: 1
Latest checkpoint's oldestCommitTsXid:0
Latest checkpoint's newestCommitTsXid:0
Time of latest checkpoint:            Mon 21 May 2018 11:54:48 AM CST
Fake LSN counter for unlogged rels:   0/1
Minimum recovery ending location:     0/0
Min recovery ending loc's timeline:   0
Backup start location:                0/0
Backup end location:                  0/0
End-of-backup record required:        no
wal_level setting:                    minimal
wal_log_hints setting:                off
max_connections setting:              300
max_worker_processes setting:         8
max_prepared_xacts setting:           0
max_locks_per_xact setting:           64
track_commit_timestamp setting:       off
Maximum data alignment:               8
Database block size:                  8192
Blocks per segment of large relation: 131072
WAL block size:                       8192
Bytes per WAL segment:                16777216
Maximum length of identifiers:        64
Maximum columns in an index:          32
Maximum size of a TOAST chunk:        1996
Size of a large-object chunk:         2048
Date/time type storage:               64-bit integers
Float4 argument passing:              by value
Float8 argument passing:              by value
Data page checksum version:           0
Data encryption:                      off
```

pg_control数据定义如下：

```
typedef struct ControlFileData
 {
     /*
      * Unique system identifier --- to ensure we match up xlog files with the
      * installation that produced them.
      */
     uint64      system_identifier;
 
     /*
      * Version identifier information.  Keep these fields at the same offset,
      * especially pg_control_version; they won't be real useful if they move
      * around.  (For historical reasons they must be 8 bytes into the file
      * rather than immediately at the front.)
      *
      * pg_control_version identifies the format of pg_control itself.
      * catalog_version_no identifies the format of the system catalogs.
      *
      * There are additional version identifiers in individual files; for
      * example, WAL logs contain per-page magic numbers that can serve as
      * version cues for the WAL log.
      */
     uint32      pg_control_version; /* PG_CONTROL_VERSION */
     uint32      catalog_version_no; /* see catversion.h */
 
     /*
      * System status data
      */
     DBState     state;          /* see enum above */
     pg_time_t   time;           /* time stamp of last pg_control update */
     XLogRecPtr  checkPoint;     /* last check point record ptr */
 
     CheckPoint  checkPointCopy; /* copy of last check point record */
 
     XLogRecPtr  unloggedLSN;    /* current fake LSN value, for unlogged rels */
 
     /*
      * These two values determine the minimum point we must recover up to
      * before starting up:
      *
      * minRecoveryPoint is updated to the latest replayed LSN whenever we
      * flush a data change during archive recovery. That guards against
      * starting archive recovery, aborting it, and restarting with an earlier
      * stop location. If we've already flushed data changes from WAL record X
      * to disk, we mustn't start up until we reach X again. Zero when not
      * doing archive recovery.
      *
      * backupStartPoint is the redo pointer of the backup start checkpoint, if
      * we are recovering from an online backup and haven't reached the end of
      * backup yet. It is reset to zero when the end of backup is reached, and
      * we mustn't start up before that. A boolean would suffice otherwise, but
      * we use the redo pointer as a cross-check when we see an end-of-backup
      * record, to make sure the end-of-backup record corresponds the base
      * backup we're recovering from.
      *
      * backupEndPoint is the backup end location, if we are recovering from an
      * online backup which was taken from the standby and haven't reached the
      * end of backup yet. It is initialized to the minimum recovery point in
      * pg_control which was backed up last. It is reset to zero when the end
      * of backup is reached, and we mustn't start up before that.
      *
      * If backupEndRequired is true, we know for sure that we're restoring
      * from a backup, and must see a backup-end record before we can safely
      * start up. If it's false, but backupStartPoint is set, a backup_label
      * file was found at startup but it may have been a leftover from a stray
      * pg_start_backup() call, not accompanied by pg_stop_backup().
      */
     XLogRecPtr  minRecoveryPoint;
     TimeLineID  minRecoveryPointTLI;
     XLogRecPtr  backupStartPoint;
     XLogRecPtr  backupEndPoint;
     bool        backupEndRequired;
 
     /*
      * Parameter settings that determine if the WAL can be used for archival
      * or hot standby.
      */
     int         wal_level;
     bool        wal_log_hints;
     int         MaxConnections;
     int         max_worker_processes;
     int         max_prepared_xacts;
     int         max_locks_per_xact;
     bool        track_commit_timestamp;
 
     /*
      * This data is used to check for hardware-architecture compatibility of
      * the database and the backend executable.  We need not check endianness
      * explicitly, since the pg_control version will surely look wrong to a
      * machine of different endianness, but we do need to worry about MAXALIGN
      * and floating-point format.  (Note: storage layout nominally also
      * depends on SHORTALIGN and INTALIGN, but in practice these are the same
      * on all architectures of interest.)
      *
      * Testing just one double value is not a very bulletproof test for
      * floating-point compatibility, but it will catch most cases.
      */
     uint32      maxAlign;       /* alignment requirement for tuples */
     double      floatFormat;    /* constant 1234567.0 */
 #define FLOATFORMAT_VALUE   1234567.0
 
     /*
      * This data is used to make sure that configuration of this database is
      * compatible with the backend executable.
      */
     uint32      blcksz;         /* data block size for this DB */
     uint32      relseg_size;    /* blocks per segment of large relation */
 
     uint32      xlog_blcksz;    /* block size within WAL files */
     uint32      xlog_seg_size;  /* size of each WAL segment */
 
     uint32      nameDataLen;    /* catalog name field width */
     uint32      indexMaxKeys;   /* max number of columns in an index */
 
     uint32      toast_max_chunk_size;   /* chunk size in TOAST tables */
     uint32      loblksize;      /* chunk size in pg_largeobject */
 
     /* flags indicating pass-by-value status of various types */
     bool        float4ByVal;    /* float4 pass-by-value? */
     bool        float8ByVal;    /* float8, int8, etc pass-by-value? */
 
     /* Are data pages protected by checksums? Zero if no checksum version */
     uint32      data_checksum_version;
 
     /*
      * Random nonce, used in authentication requests that need to proceed
      * based on values that are cluster-unique, like a SASL exchange that
      * failed at an early stage.
      */
     char        mock_authentication_nonce[MOCK_AUTH_NONCE_LEN];
 
     /* CRC of all above ... MUST BE LAST! */
     pg_crc32c   crc;
 } ControlFileData;
```

pg_control文件首次创建是在src/backend/access/transam/xlog.c中的void [BootStrapXLOG](https://doxygen.postgresql.org/xlog_8c.html#aa67c99e001f4fb4a3e6f5ae99d7efddf)(void) 完成。

BootStrapXLOG(void)函数在系统安装时仅执行一次，负责创建pg_control文件以及初始化XLOG文件。



Database system identifier（数据库系统标识符，内部提示为sysid），用于唯一识别Database Cluster，启动、备份或者恢复等过程中会校验pg_control中的Database system identifier与wal文件中的sysid是否相同。

sysid生成的算法为：

```
     uint64 sysidentifier;
     
     /*
      * Select a hopefully-unique system identifier code for this installation.
      * We use the result of gettimeofday(), including the fractional seconds
      * field, as being about as unique as we can easily get.  (Think not to
      * use random(), since it hasn't been seeded and there's no portable way
      * to seed it other than the system clock value...)  The upper half of the
      * uint64 value is just the tv_sec part, while the lower half contains the
      * tv_usec part (which must fit in 20 bits), plus 12 bits from our current
      * PID for a little extra uniqueness.  A person knowing this encoding can
      * determine the initialization time of the installation, which could
      * perhaps be useful sometimes.
      */
     gettimeofday(&tv, NULL);
     sysidentifier = ((uint64) tv.tv_sec) << 32;
     sysidentifier |= ((uint64) tv.tv_usec) << 12;
     sysidentifier |= getpid() & 0xFFF;
```

pg_controldata显示sysid时，基于跨平台展示的考虑，采用char进行了转换：

```
     /*
      * Format system_identifier and mock_authentication_nonce separately to
      * keep platform-dependent format code out of the translatable message
      * string.
      */
     snprintf(sysident_str, sizeof(sysident_str), UINT64_FORMAT,
     ControlFile->system_identifier);
```



