
# MySQL 数据备份与还原

## 1. 概述

本主要介绍 MySQL 逻辑备份的两种主流方案：官方原生的 **mysqldump** 和多线程高性能工具 **mydumper**。同时针对备份过程中可能出现的“静默失败”、IO 瓶颈及锁表问题，附加提供监控与应对方案，监控方案需要配置prometheus来管理，这里只是提供一下思路方案，具体配置不做介绍。

---

## 2. 备份工具详解

### 2.1 mysqldump (官方原生)

mysqldump是MySQL数据库自带的逻辑备份工具，属于热备工具。它的备份结果是根据设置的参数将数据库中的信息通过生成创建库、表等对象以及对应表的insert语句组成。

mysqldump 参数选项特别多，可以通过mysqldump --help 查看对应的参数及说明

```bash
[root@testdb ~]# mysqldump --help
mysqldump  Ver 10.13 Distrib 5.7.25-28, for Linux (x86_64)
Copyright (c) 2009-2019 Percona LLC and/or its affiliates
Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Dumping structure and contents of MySQL databases and tables.
Usage: mysqldump [OPTIONS] database [tables]
OR     mysqldump [OPTIONS] --databases [OPTIONS] DB1 [DB2 DB3...]
OR     mysqldump [OPTIONS] --all-databases [OPTIONS]
```

适用于数据量较小（建议 < 50GB）的场景，生成的 SQL 兼容性好，便于迁移。

#### 常用参数详解

| **参数**                | **说明**                                                            | **推荐场景**         |
| ----------------------- | ------------------------------------------------------------------- | -------------------- |
| `--single-transaction`  | 在开始备份前开启事务，保证数据一致性且不锁表（仅 InnoDB）。         | **必选**             |
| `--master-data=2`       | 记录备份时刻的 Binlog 位置（以注释形式），用于搭建从库。            | 搭建主从时必选       |
| `default-character-set` | 设置字符集，建议指明字符集。                                        | 完整备份建议开启     |
| `databases`             | 指定多个库备份参数。                                                | 多库备份开启         |
| `all-databases`         | 全库备份参数。                                                      | 全库备份开启         |
| `--quick`               | 逐行读取表数据，而不是一次性加载到内存，防止 OOM。                  | 大表备份必选         |
| `--routines (-R)`       | 备份存储过程和函数。                                                | 完整备份建议开启     |
| `--triggers`            | 备份触发器。                                                        | 完整备份建议开启     |
| `--events (-E)`         | 备份定时任务。                                                      | 完整备份建议开启     |
| `--hex-blob`            | 将二进制字段导出为十六进制，防止乱码。                              | 含 Blob/Binary 字段  |
| `--set-gtid-purged`     | 在 GTID 模式下，如果仅做普通恢复而非建立复制，建议关闭(OFF)。       | 数据迁移/本地测试    |
| `no-data(-d)`           | 只备份表结构，不包含数据。                                          | 只迁移表结构         |
| `no-create-info(-t)`    | 只备份数据，不备份建表信息。                                        | 只迁移表数据         |
| `flush-logs`            | 备份完成后切换日志。                                                | 日志输出级别较高     |
| `flush-privileges`      | 备份完成后刷新权限。                                                | 权限要求较高         |
| `where`                 | 指定条件，例如每张表导出1000行的记录或者 导出每张表id<=10的记录等。 | 只要求备份示例数据等 |
| `skip-add-drop-table`   | 不生成删除表的语句。                                                | 无需生成删除表语句   | 

#### 标准备份命令示例

```Bash
# 全量备份
mysqldump -u root -p'Password' -h 127.0.0.1 -P 3306 \
  --single-transaction --master-data=2 --quick \
  --routines --triggers --events --hex-blob \
  --all-databases | gzip > /backup/full_backup_$(date +%F).sql.gz
```

#### 2.1.1  备份指定表

mysqldump可以备份指定的单个表或指定库的多个表，例如备份testdb库的test1表的表结构和数据

```Bash
# 备份testdb库的test1表
mysqldump -uroot -p --master-data=2 --default-character-set=utf8 --single-transaction  testdb  test1 > test1.sql
```

#### 2.1.2.  备份单个数据库

mysqldump可以备份指定的数据库，可以是单个库也可以是多个库，备份单库示例如下：

```Bash
# 备份整个testdb库
mysqldump -uroot -p  --master-data=2 --default-character-set=utf8 --single-transaction testdb > testdb.sql
```

#### 2.1.3  备份多个库

备份多个数据库可以用如下命令

```Bash
# 备份多个testdb库
mysqldump -uroot -p --master-data=2 --default-character-set=utf8 --single-transaction --databases testdb1 testdb2  > mul_db.sql
```

#### 2.1.4  备份所有的数据库  

如果想备份所有的数据库，可以使用如下命令：

```Bash
# 备份所有数据库
mysqldump -uroot -p --master-data=2 --default-character-set=utf8 --single-transaction --all-databases > all_db.sql
```

> 注：备份中没有information\_schema、performance\_schema、sys库的信息（MySQL5.7及以上版本）

#### 2.1.5  其他情况

实际使用中可能还会遇到只备份表结构、只备份数据，需要备份存储过程及事件等需求，相应参数如下：

```Bash
--no-data    　　　　只备份表结构，不包含数据，可以简写为 -d
--no-create-info  　只备份数据，不备份建表信息，也可以简写为-t
--flush-logs  　　　 备份完成后切换日志
--flush-privileges  备份完成后刷新权限
--where       　　　 指定条件，例如每张表导出1000行的记录或者 导出每张表id<=10的记录等，可以参考历史文章查看示例
--skip-add-drop-table 不生成删除表的语句
```

备份全部数据库，包含触发器、事件、存储过程，同时刷新日志及权限的示例：

```Bash
mysqldump -uroot -p --master-data=2 --default-character-set=utf8 --routines --triggers --events --flush-logs --flush-privileges --single-transaction --all-databases >backup.sql
```

#### 2.1.6 还原数据

还原数据直接使用source命令导入sql文件就可以，但是需要注意的是一下两点，防止数据误删等

*a) 还原命令使用起来比较方便，但是实际生产环境中还原数据时不建议直接还原至目标表里（尤其处理误删除恢复数据时），而是建议先还原至其他实例或其他库里，确认无误后再将需要还原的记录导入至目标表里;*

*b) 要警惕备份文件中是否有删除库或删表的指令，否则如果选择在同一实例中还原即使选择了临时恢复的库，而备份文件里有use db；及drop table的语句，则会将目标表全部删掉。*

#### 2.1.7 搭配 pv (Pipe Viewer) 使用

 **`pv`** 是一个非常强大的命令行工具，用于**监控通过管道（Pipe）的数据流进度**，并且可以进行**限速**。

**注意：**

- `pv` **非常适合** `mysqldump`（因为它是单线程数据流）。
    
- `pv` **不适合** 直接用于 `myloader`，因为 `myloader` 是直接读取文件夹里的多个文件，不走管道流。

#####  核心场景一：带进度条的导入 (配合 mysqldump/mysql)

当你有一个巨大的 `backup.sql` 要导入 MySQL 时，原本的 `mysql < backup.sql` 是没有反应的。

**使用 pv 监控导入进度：**

```Bash
# 语法：pv [文件名] | mysql ...
# -W: 遇到错误不等待用户确认
pv /backup/full_backup.sql | mysql -u root -p'Pass' -D my_db
```

**输出效果：**

```Plaintext
1.25GiB 0:02:15 [15.2MiB/s] [===============>        ] 45% ETA 0:03:00
```

- **1.25GiB**: 已处理数据量
    
- **15.2MiB/s**: 当前导入速度 (IO吞吐)
    
- **45%**: 进度百分比
    
- **ETA**: 预计剩余时间
    
##### 核心场景二：导出时限速 (防止 IO 跑满)

如果在业务高峰期做 `mysqldump`，为了不把磁盘 IO 打满导致业务卡顿，可以使用 `pv` 进行**限速**。

**参数详解：**

- `-L RATE`: 限制最大传输速率 (如 `10m`, `500k`)。
    
- `-q`: 安静模式（不输出进度条，只用于限速时很有用）。
    
**命令示例：**

```Bash
# 限制导出速度最大为 20MB/s，并压缩
mysqldump --single-transaction --all-databases \
  | pv -L 20m \
  | gzip > /backup/slow_backup.sql.gz
```

这样，原本可能跑 200MB/s 把磁盘读废的操作，就被压制在 20MB/s，对业务影响极小。

#### 4. 核心场景三：解压导入时的进度监控

如果是 `.gz` 压缩包导入，`pv` 应该放在解压之前还是之后？通常放在前面看读取进度。

```Bash
# 监控 gzip 读取进度 -> 解压 -> 导入数据库
pv /backup/backup.sql.gz | gunzip | mysql -u root -p'Pass' -D my_db
```

#### 2.2 mydumper & myloader (多线程高性能)

适用于大数据量（> 50GB）场景。它支持多线程并行导出（表级或行级切分），速度远快于 mysqldump。

##### mydumper (备份) 常用参数

| **参数**              | **说明**                                           | **备注**            |
| --------------------- | -------------------------------------------------- | ------------------- |
| `-t, --threads`       | 并发线程数。建议设置为 CPU 核心数或核心数的 2 倍。 | **核心参数**        |
| `-r, --rows`          | 将大表切分为多少行一个文件（Chunk）。              | 建议 500000~1000000 |
| `-B, --database`      | 指定备份的数据库。                                 |                     |
| `-c, --compress`      | 压缩输出文件（.gz）。                              | 节省磁盘空间        |
| `-L, --logfile`       | 指定日志文件路径。                                 | **监控关键**        |
| `-v, --verbose`       | 日志详细级别 (0-3)。3 为最详细。                   | **监控关键**        |
| `-o, --output`        | 指定sql输出目录                                    | 必备参数            |
| `--long-query-guard`  | 长查询保护，超时则退出备份，避免阻塞业务。         | 默认 60s            |
| `--kill-long-queries` | 杀掉长查询以保证备份进行（慎用）。                 | 生产环境慎用        |

###### 标准备份命令示例

开启8个线程，备份my_db数据库，将大表切分为500000行一个文件，同时压缩输出文件节省磁盘空间，并且指定输出日志文件，设置日志等级为3：

```Bash
mydumper -u root -p 'Password' -h 127.0.0.1 -P 3306 \
  -t 8 -r 500000 -c \
  -B my_db \
  -o /backup/mydumper_$(date +%F) \
  -L /backup/logs/backup_$(date +%F).log \
  -v 3
```

##### myloader (恢复) 常用参数

| **参数**   | **长参数**                  | **含义**                                 | **建议值/说明**                                            |
| ---------- | --------------------------- | ---------------------------------------- | ---------------------------------------------------------- |
| `-d`       | `--directory`               | **必选**，备份文件所在的目录。           | mydumper 导出的那个目录。                                  |
| `-t`       | `--threads`                 | 恢复使用的并发线程数。                   | 建议与 CPU 核数一致。过高会导致磁盘 IO 瓶颈。              |
| `-o`       | `--overwrite-tables`        | **慎用**，如果表存在，先 `DROP` 再创建。 | 全量恢复时通常需要加上。                                   |
| `-B`       | `--database`                | 指定恢复到哪个数据库。                   | 如果备份了多个库，但只想恢复其中一个时使用。               |
| `-s`       | `--source-db`               | 源数据库名。                             | 配合 `-B` 使用，用于跨库恢复（把 A 库的数据恢复到 B 库）。 |
| `-v`       | `--verbose`                 | 日志详细级别 (0-3)。                     | **务必设为 3**，否则两眼一抹黑。                           |
| `-L`       | `--logfile`                 | 日志文件路径。                           | 将输出写入文件，方便后续 `tail -f` 查看。                  |
| `-r`       | `--rows`                    | (不常用) 批量插入的行数。                | 默认即可。                                                 |
| `-q`       | `--queries-per-transaction` | 每个事务包含的查询数。                   | 默认 1000。调大可提高速度但增加主从延迟。                  |
| `--resume` | **断点续传**。              | 尝试从上次中断的地方继续恢复。           |                                                            |

###### 标准恢复命令示例

开启4个线程，指定备份目录为/backup/20260114，开启-o参数，添加drop语句，并且指定日志输出路径，设置日志等级为3，任务为后台执行，这个后台执行一般为默认选择，因为执行时间都比较长：

```Bash
# 建议放入 nohup 或 screen/tmux 中执行，防止终端断开
nohup myloader \
  -u root -p 'YourPassword' -h 127.0.0.1 -P 3306 \
  -d /backup/20260114 \
  -t 4 \
  -o \
  -v 3 \
  -L /backup/logs/restore_20260114.log &
```

###### 查看数据导入进度情况

由于 `myloader` 是多文件并行恢复，无法像单文件那样看“百分比”，但我们可以通过以下方式监控：

**方法 A：实时查看日志 (最推荐)** 因为开启了 `-v 3`，日志会打印每个文件的恢复开始和结束。

```Bash
# 查看实时日志
tail -f /backup/logs/restore_20260114.log

# 输出示例：
# Thread 1 restoring database.table.00001.sql
# Thread 2 restoring database.table.00005.sql
# ...
```

**方法 B：查看 Processlist** 登录数据库查看当前正在执行的线程状态。

```SQL
SHOW PROCESSLIST;
-- 你会看到多个线程正在执行 INSERT 或 CREATE TABLE 操作
```

**方法 C：监控已完成的文件数 (粗略进度)** mydumper 目录里主要是 `.sql` 文件，可以通过统计日志中 "Finished" 的数量来估算进度。

```Bash
# 统计已完成的任务块数量
grep "Finished work" /backup/logs/restore_20260114.log | wc -l
```

---


## 3. 备份过程日志与中断监控方案

由于 `mydumper` 在默认情况下输出较少，且我们需要捕捉备份是否成功，必须通过**日志重定向**与**状态码判断**来实现。

### 3.1 mysqldump 日志捕获

`mysqldump` 的错误信息输出到 `stderr`，需要重定向。

```Bash
#!/bin/bash
BACKUP_DIR="/data/backup"
LOG_FILE="${BACKUP_DIR}/mysqldump_$(date +%F).log"
DATE=$(date +%F_%H-%M-%S)

echo "[${DATE}] Backup Started" > ${LOG_FILE}

# 2>> 将错误日志追加到文件
mysqldump -u root -p'Pass' --all-databases --single-transaction \
  > ${BACKUP_DIR}/db_${DATE}.sql 2>> ${LOG_FILE}

if [ $? -eq 0 ]; then
    echo "[$(date +%F_%H-%M-%S)] Backup Success" >> ${LOG_FILE}
else
    echo "[$(date +%F_%H-%M-%S)] Backup FAILED" >> ${LOG_FILE}
    # 此处可调用告警接口
fi
```

### 3.2 mydumper 日志适配 (解决无输出问题)

`mydumper` 需要利用 `-v 3` 和 `-L` 参数来生成详细日志，并通过脚本判断最终状态。

**推荐的 Wrapper 脚本 (`run_mydumper.sh`):**

```Bash
#!/bin/bash

BACKUP_ROOT="/data/backup"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="${BACKUP_ROOT}/${TIMESTAMP}"
LOG_FILE="${BACKUP_ROOT}/logs/backup_${TIMESTAMP}.log"

mkdir -p ${BACKUP_DIR}
mkdir -p ${BACKUP_ROOT}/logs

echo "------------------------------------------------" >> ${LOG_FILE}
echo "Job started at $(date)" >> ${LOG_FILE}

# 执行备份
# -v 3: 输出详细信息 (Info级别)
# -L: 指定日志文件路径，解决无输出问题
mydumper \
  -u root -p 'YourPass' -h 127.0.0.1 \
  -t 4 \
  -o ${BACKUP_DIR} \
  -L ${LOG_FILE} \
  -v 3 

EXIT_CODE=$?

echo "------------------------------------------------" >> ${LOG_FILE}

if [ ${EXIT_CODE} -eq 0 ]; then
    # 检查 metadata 文件是否存在，这是 mydumper 成功的核心标志
    if [ -f "${BACKUP_DIR}/metadata" ]; then
        echo "Status: SUCCESS" >> ${LOG_FILE}
        echo "Job finished at $(date)" >> ${LOG_FILE}
        
        # [Prometheus] 将成功状态写入 Textfile Collector
        echo 'mysql_backup_status{type="mydumper"} 1' > /node_exporter/textfile_collector/mysql_backup.prom
        echo "mysql_backup_last_success_timestamp $(date +%s)" >> /node_exporter/textfile_collector/mysql_backup.prom
    else
        echo "Status: FAILED (Metadata missing)" >> ${LOG_FILE}
        echo 'mysql_backup_status{type="mydumper"} 0' > /node_exporter/textfile_collector/mysql_backup.prom
    fi
else
    echo "Status: FAILED (Exit Code: ${EXIT_CODE})" >> ${LOG_FILE}
    # 出现错误，打印最后几行日志辅助排查
    tail -n 10 ${LOG_FILE}
    
    # [Prometheus] 写入失败状态
    echo 'mysql_backup_status{type="mydumper"} 0' > /node_exporter/textfile_collector/mysql_backup.prom
fi
```

### 3.3 备份中断处理 SOP

如果监控发现备份中断（Exit Code != 0 或日志中有 Error）：

1. **检查磁盘空间：** 备份通常会瞬间占用大量 IO 和 空间。`df -h` 查看备份目录是否已满。
    
2. **检查 OOM (内存溢出)：** `dmesg | grep -i 'killed'` 查看是否因为 `mysqldump` 或 `mydumper` 占用过多内存被系统杀掉。
    
3. **检查网络超时：** 如果是远程备份，检查 `net_read_timeout` 和 `net_write_timeout` 设置。
    
4. **操作建议：**
    
    - **不要**尝试断点续传（逻辑备份难以做到数据一致性的断点续传）。
        
    - 删除损坏的/不完整的备份目录。
        
    - 解决 IO 或 空间问题后，**重新发起** 完整备份。
        

---

## 4. 性能影响应对与 Prometheus 监控方案

备份是一个高 IO、高 CPU 的操作，且在开始阶段（FTWRL）会有短暂的全局锁（尽管 `--single-transaction` 尽量减少了锁，但获取元数据锁 MDL 依然存在）。

### 4.1 锁表与 IO 瓶颈应对策略

1. **在从库 (Slave) 备份：**
    
    - **方案：** 永远优先选择在从库进行备份。
        
    - **操作：** 可以在备份前停止 SQL 线程 (`STOP SLAVE SQL_THREAD`) 防止数据变动影响备份一致性（可选），备份完再开启。
        
2. **IO 限制 (Throttling)：**
    
    - **Linux 层级：** 使用 `ionice -c2 -n7` 和 `nice -n19` 降低备份进程的 IO 和 CPU 优先级。
        
    - **mysqldump:** 配合 `pv` 命令限制流速 `mysqldump ... | pv -L 10m > backup.sql`。
        
    - **mydumper:** 减少 `-t` 线程数。
        
3. **长事务阻塞 (MDL Lock)：**
    
    - 如果业务有长事务正在执行，备份工具无法获取 MDL 锁，会卡在 `Waiting for table metadata lock`。
        
    - **方案：** 使用 `--long-query-guard` (mydumper) 自动放弃备份，防止拖垮业务数据库。
        

### 4.2 Prometheus 监控配置

我们需要监控两个维度：**备份任务本身的状态** 和 **备份对系统的影响**。

#### 4.2.1 监控指标 (Exporter)

1. **Node Exporter:** 监控磁盘 IO 和 CPU。
    
2. **MySQL Exporter:** 监控数据库锁和连接数。
    
3. **Textfile Collector (自定义):** 监控备份脚本的执行结果（如 3.2 中所示）。
    

#### 4.2.2 关键告警规则 (Prometheus Rules)

**1. 备份任务失败告警**

```YAML
- alert: MySQLBackupFailed
  expr: mysql_backup_status == 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "MySQL Backup Failed on {{ $labels.instance }}"
    description: "Please check logs in /data/backup/logs/"
```

**2. 备份时效性告警 (超过 24h 未成功)**

```YAML
- alert: MySQLBackupOutdated
  expr: (time() - mysql_backup_last_success_timestamp) > 86400
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "No successful backup in last 24h"
```

**3. 备份导致的高 IO 等待 (IO瓶颈)**

如果在备份期间 IO Wait 过高，说明需要限制备份速度或增加硬件。

```YAML
- alert: HighIOWaitDuringBackup
  # 监控 iowait > 20% 且 备份状态为正在进行(假设你有指标标记备份开始)
  expr: rate(node_cpu_seconds_total{mode="iowait"}[5m]) * 100 > 20
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "High IO Wait detected on {{ $labels.instance }}"
```

**4. 锁等待监控 (MDL 锁)**

监控备份开始时是否导致了大量的锁等待。

```YAML
- alert: MySQLLockWaitHigh
  expr: mysql_global_status_threads_running > 50
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: "Possible Metadata Lock storm detected"
```

### 4.3 自动化应对 (Alertmanager Webhook)

当 Prometheus 触发 `HighIOWaitDuringBackup` 告警时，可以通过 Webhook 调用 Ansible 或 Shell 脚本执行以下降级操作：

1. **Renice:** 自动降低备份进程 PID 的优先级。
    
2. **Pause:** 针对 mydumper，虽不支持动态暂停，但可以考虑临时停止从库同步以减少磁盘争用。
    
3. **Kill:** 如果严重影响主库（如 Threads_running 飙升），脚本自动 Kill 掉备份进程以保住业务。

***注：***

