---
title: "Langfuse 吃满磁盘紧急修复"
date: 2026-09-01T19:09:57+08:00
description: ""
menu:
  sidebar:
    name: "Langfuse 吃满磁盘紧急修复"
    identifier: tips-langfuse
    parent: tips
    weight: 100
tags: ["Langfuse", "Linux"]
categories: ["Tips"]
---

记录一次生产环境 Langfuse Self-Hosted 吃满服务器磁盘的排查和修复经历（GPT 牛逼）。

<!--more-->

## 故障背景

发现服务器磁盘满了，服务器上有一个轻量的 AI 服务，还部署了一个 Self-Hosted 的 Langfuse 来观测 AI 服务。

因为 AI 服务很轻量，数据库用的是云端的，本地日志不大可能写满磁盘，所以首先怀疑是 Langfuse 相关的服务吃满了磁盘。

## 进入容器

Langfuse 是用 Podman Compose 部署的，所以想着先运行 podman 命令查看状态，结果直接报错：

```text
... failed to unmarshal existing healthcheck results in ...
```

看来是磁盘空间满了导致 Podman healthcheck 相关的文件也坏了。

Langfuse 背后有很多容器，其中数据相关的服务就包括了 Postgres、ClickHouse、MinIO 和 Redis。
这里最可疑的是 ClickHouse，因为大量的 trace 都在这里。

在 Podman CLI 无法使用的情况下检查进入容器检查，可以先查找到容器的 PID，然后用 `nsenter` 命令：

```bash
sudo nsenter \
  -t <PID> \
  --user \
  --mount \
  --uts \
  --ipc \
  --net \
  --pid \
  --root \
  --wd \
  /bin/bash
```

## 连接数据库

成功进入容器后，运行 `clickhouse-client` 试图连接数据库检查，但得到报错：

```text
std::exception. Code: 1001, type: std::__1::filesystem::filesystem_error, e.what() = filesystem error: in create_directories: No space left on device ["/home/deployer"]
Total space: 58.09 GiB
Available space: 0.00 B
Total inodes: 3.87 million
Available inodes: 2.74 million
Mount point: /
Filesystem: overlay (version 26.5.1.882 (official build))
```

ClickHouse Client 在磁盘满时无法启动，原因是 `clickhouse-client` 会创建本地历史记录，可以用 `--history_file=/dev/null` 来临时绕过。

成功连接数据库后，尝试直接给 Langfuse 设置 30 天的 TTL（因为默认是永久保存的，实际上我们用不到）:

```sql
ALTER TABLE traces MODIFY TTL timestamp + INTERVAL 30 DAY DELETE;
ALTER TABLE observations MODIFY TTL start_time + INTERVAL 30 DAY DELETE;
ALTER TABLE scores MODIFY TTL timestamp + INTERVAL 30 DAY DELETE;
ALTER TABLE event_log MODIFY TTL created_at + INTERVAL 30 DAY DELETE;
ALTER TABLE blob_storage_file_log MODIFY TTL created_at + INTERVAL 30 DAY DELETE;
```

结果报错：

```text
Received exception from server (version 26.5.1):
Code: 75. DB::Exception: Received from localhost:9000. DB::ErrnoException. DB::ErrnoException: Cannot write to file /var/lib/clickhouse/store/c1a/c1a0925e-d846-4a1d-af9d-98331fe92460/traces.sql.tmp: , errno: 28, strerror: No space left on device
Total space: 58.09 GiB
Available space: 0.00 B
Total inodes: 3.87 million
Available inodes: 2.74 million
Mount point: /var/lib/clickhouse
Filesystem: /dev/vda2. (CANNOT_WRITE_TO_FILE_DESCRIPTOR)
```

即使 ALTER TABLE 只是修改 TTL，ClickHouse 仍需要写表元数据临时文件，所以 `ALTER TABLE` / `DROP PARTITION` 和 `DELETE` 都可能无法正常执行。

## 分析磁盘占用

既然无法执行命令，先退出 ClickHouse，分析一下 ClickHouse 的实际磁盘占用（其实这里应该直接查 ClickHouse 表的，我当时以为修改不行查询也不行）。

通过 `/proc/<PID>/root` 可以直接从宿主机访问容器文件系统：

```bash
du -axh /proc/<PID>/root/var/lib/clickhouse \
  /proc/<PID>/root/var/log/clickhouse-server 2>/dev/null |
sort -nr |
head -30
```

发现 `/var/lib/clickhouse/store` 占了 27 GB，而其中两个最大的 ClickHouse Store UUID 里面有 `timestamp_ns.bin`、`event_time_microseconds.bin` 和 `message.bin`，于是推测这些是 `system.trace_log` 和 `system.text_log`，而不是 Langfuse 业务 trace。

之后尝试直接删除 ClickHouse 的日志，来腾出空间执行其他命令：

```bash
find /proc/<PID>/root/var/log/clickhouse-server \
  -maxdepth 1 \
  -type f \
  -name "*.log" \
  -print \
  -exec truncate -s 0 {} \;
```

当时确实成功腾出了一些空间，于是再次试图修改 TTL，但执行到一半又因为磁盘满而报错。

这时候再回到 `clickhouse-client`，执行查询：

```sql
SELECT
    database,
    name,
    uuid,
    engine,
    formatReadableSize(total_bytes) AS size
FROM system.tables
WHERE toString(uuid) IN
(
    ...
);
```

发现它们的确是 `text_log` 和 `trace_log`。
因为 Langfuse 根本不用这些，可以放心删除。

```sql
TRUNCATE TABLE system.trace_log;
TRUNCATE TABLE system.text_log;
```

再次尝试修改 TTL，发现依然失败，很遗憾磁盘并没有立刻空出来。

## 修改保留块（Reserved Blocks）

执行 `df -h` 可以发现我们用的 Ext4 文件系统默认给了 5% 的保留块：

```text
Size  Used  Avail  Use%
59G   56G   0      100%
```

为了给 ClickHouse 腾空间执行更多操作，现在只能先使用一部分保留空间了。

```bash
sudo tune2fs -m 1 /dev/vda2
```

赶紧再次回到 `clickhouse-client`，执行相关修改，这次成功执行完毕。

## 恢复保留块

适当等一会，等 ClickHouse 数据清理完成。

再次执行 `df -h`，发现磁盘已有足够余量后，再恢复 ext4 默认保留块：

```bash
sudo tune2fs -m 5 /dev/vda2
```

## 修复 Podman 报错

定位到损坏的容器对应 Podman rootless runtime 的路径，直接把损坏的 healthcheck 文件移走：

```bash
mv "/path/to/healthcheck.log" "/tmp"
```

重新执行 `podman ps -a`，一切正常。

## 修改 ClickHouse 配置

为了防止以后再次出现不必要的 system log 写满磁盘，可以直接关闭相关功能，因为 Langfuse 用不到。

创建配置 `clickhouse-config/disable-system-logs.xml`：

```xml
<clickhouse>
    <trace_log remove="1"/>
    <text_log remove="1"/>
    <opentelemetry_span_log remove="1"/>
    <asynchronous_metric_log remove="1"/>
    <metric_log remove="1"/>
    <latency_log remove="1"/>
</clickhouse>
```

修改 `docker-compose.yml` 挂载配置：

```yaml
services:
  clickhouse:
    volumes:
      - ./clickhouse-config/disable-system-logs.xml:/etc/clickhouse-server/config.d/disable-system-logs.xml:ro,Z
```

## 配置 MinIO

同样的，给 MinIO 的 events 设置 30 天 Lifecycle：

```bash
podman exec -it <MinIO 容器> bash
mc alias set local \
  http://127.0.0.1:9000 \
  <user> \
  <password>
mc ilm rule add \
  local/langfuse \
  --prefix "events/" \
  --expire-days 30
```

## 重启服务

重启服务，再回到 `clickhouse-client` 清理相关数据表：

```sql
SET max_table_size_to_drop = 0;

TRUNCATE TABLE system.trace_log;
TRUNCATE TABLE system.text_log;
```

确认所有服务正常运行，至此，因为 ClickHouse 吃满服务器磁盘的紧急修复完毕。

Langfuse 的企业版是可以设置 30 天自动过期的，而 Self Host 版本只能通过手动改 ClickHouse 表来实现，之前一直以为轻量服务用不了多少空间，现在看来不能抱有侥幸心理（服务器硬盘也选小了，才 60 G）。

## 题外话

因为不可能把生产环境直接交给 AI（别问我为什么），所以我是把场景和具体的命令输出（过滤掉隐私内容）给网页版 GPT，让它来分析并给我指示，实际执行命令前也要确定命令不会破坏生产环境。

事实证明，只要给对信息，少走弯路，GPT 对各种命令和日志的分析比我强多了，GPT 牛逼！
我如果挨个去查文档查命令，绝对不可能在 1 小时内完成这些工作，而且 GPT 也会提供相关的文档或者论坛链接，很方便直接去查看原文，确保它给的建议和命令是正确的。
