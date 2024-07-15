---
title: 当你要搭建一个 Redis 压测环境
date: 2024-07-16 01:19:01
tags:
- Redis
- Linux
categories:
- Checklist
---

我在年纪尚轻经验尚浅的时候，接到过一个任务。有个功能需要测试性能，设计上大量使用 Redis，我需要搭建一个 Redis 的性能测试环境，来找出最佳配置。对于当时只会用 API 实现需求的我来说，这显然不是个简单的任务。我面对的是许多 Redis 配置选项的作用和原理，以及这些配置选项背后的操作系统功能。一连好几天，我迷失在各种晦涩的英文文档和大量可疑的中文博客里，如同依靠一块磁铁就想在亚马逊丛林里找出一条可以复用的路线。那个任务我完成得并不好，最后是靠着一步一步测试摸索出一个似乎还行的组合。我念了一串不明所以的咒语，得到了一只长得像鸽子的白色鸟类。

时至今日，我又经历了许多次丛林历险。虽然丛林仍是丛林，没有变成灌木丛，但我多少掌握了一些「野外生存技巧」。我总结了一个 Checklist，记录了关于 Redis 运维配置的一些思路，作为曾经丛林历险生活的纪念品。

## 配置 checklist

Linux 环境调整

- [ ]  允许内核超量使用内存 `vm.overcommit_memory=1` 。
- [ ]  降低内存 swap 概率 `swappiness`。
- [ ]  禁用内存透明大页 THP。
- [ ]  增大进程同时打开文件数量。
- [ ]  增大 TCP 全连接队列长度。
- [ ]  配置网络防火墙。
- [ ]  使用非 root 用户启动。
- [ ]  （可选）定时备份数据。

Redis 服务配置

- [ ]  设置最大内存。
- [ ]  开启持久化。
- [ ]  设置慢日志阈值。
- [ ]  重命名高危命令防止误操作。
- [ ]  修改默认端口防止扫描。
- [ ]  （可选）调整 hash、zset 等数据类型的 listpack （ziplist）使用条件。
- [ ]  监控 bigkey。
- [ ]  监控热点 Key。
- [ ]  启用延迟监控功能。
- [ ]  （可选）监控内存碎片化。

高并发场景

- [ ]  增加最大连接客户端数量。
- [ ]  增加 tcp-backlog。
- [ ]  读写分离。
- [ ]  （可选）增加 I/O 子线程。
- [ ]  （可选）Redis 绑定 CPU 核心，提高缓存命中率。

## Linux 环境配置

### 内存设置

- **允许内核超量使用内存 `vm.overcommit_memory=1` 。**

    RDB 持久化和 AOF 重写需要 fork 子进程，当可用内存（物理+swap）不足时，fork 会失败。调整为 1 后，允许内存不足时 fork。

    ```bash
    # 查看
    cat /proc/sys/vm/overcommit_memory
    # 临时修改，重启后还原
    sysctl vm.overcommit_memory=1
    # 持久修改，重启后生效
    echo "vm.overcommit_memory=1" >> /etc/sysctl.conf
    ```

    💡 /proc 不是一个真实的目录，因此对其的修改不持久，系统重启后会还原。类似 `sysctl vm.overcommit_memory=1` 的效果。

    💡 /etc/sysctl.conf 文件用于设置 Linux 内核参数，系统重启后才生效，可以用 `sysctl -p` 命令加载而无须重启。参考[How-to-modify-sysctl-settings](https://support.cpanel.net/hc/en-us/articles/360053737493-How-to-modify-sysctl-settings)

    重定向提示权限不够时，用 `tee` 命令。

    ```bash
    echo "vm.overcommit_memory=1" | sudo tee -a /etc/sysctl.conf > /dev/null
    ```

    **配合 `maxmemory` 使用**，设置 70%～80% 物理内存。

- **降低内存 swap 概率 `swappiness`。**

    swap 影响 Redis 命令性能。`swappiness` 越大，swap 概率越高。最小值为 0，但 Linux 3.5+ 时 `swappiness=0` 代表内存不足时会杀死用户进程（包括 Redis） 回收内存，应该避免。取值策略：尽量小，宁愿用 swap 也不能杀死 redis 进程。

    - ≤ Linux 3.4，`swappiness` 取 0。
    - Linux 3.5+，`swappiness`  取 1。

    ```bash
    # 临时修改，重启后还原
    echo 1 > /proc/sys/vm/swappiness
    # 临时修改，重启后还原
    sysctl vm.swappiness=1
    # 持久修改，重启后生效
    echo "vm.swappiness=1" >> /etc/sysctl.conf
    ```

    相关命令

    ```bash
    uname -a # 查看内核版本
    cat /proc/version # 查看内核版本
    free -h # 查看系统 swap
    vmstat 1 # 每隔 1s 输出一次，关注 si so 两列
    cat /proc/{pid}」/smaps | grep -i swap # 查看 pid 进程的 swap 使用
    sysctl -p # 立即加载 /etc/sysctl.conf 文件
    ```

- **禁用内存透明大页 THP。**

    开启 THP 能提升内存访问效率，但会降低内存时复制的性能。根据 Redis 作者这篇 [Linux 内核如何影响 Redis 性能的文章](http://antirez.com/news/84)，Redis 依靠子进程实现 RDB 持久化和 AOF 重写，开启 THP 会增加重写期间父进程的内存消耗，也会拖慢重写期间命令执行速度。

    ```bash
    # 检查 THP
    cat /sys/kernel/mm/transparent_hugepage/enabled
    # 临时禁止 THP
    echo never > /sys/kernel/mm/transparent_hugepage/enabled
    echo never > /sys/kernel/mm/transparent_hugepage/defrag
    ```

    持久修改 THP 需要根据 Linux 发行版采取不同设置。

    关于 THP 的详细接收，可以参考下面两篇文章。

    [6.3. 禁用 Transparent Huge Pages 功能 | Red Hat Product Documentation](https://docs.redhat.com/zh_hans/documentation/red_hat_directory_server/12/html/tuning_the_performance_of_red_hat_directory_server/proc_disabling-the-transparent-huge-pages-feature_assembly_tuning-resource-limits)

    [我们为什么要禁用 THP 丨TiDB 应用实践 | PingCAP](https://cn.pingcap.com/blog/why-should-we-disable-thp/)

    💡 常用数据库 MySQL、MongoDB 都会建议关闭 THP。

### 网络设置

- **增大进程同时打开文件数量。**

    Linux 秉持「一切皆文件」的思想，将网络连接（套接字）也对应为一个文件。同时，Linux 限制每个进程能同时打开的文件描述符数量，也就限制了进程的最大连接数。一旦超过上限，会报 error: too many open files 错误。

    通过 `ulimit -n` 可以查看进程的文件描述符限制，通常为 1024。通过 `ulimit` 命令设置阈值，有两种阈值。

    - soft 代表警告阈值，可以超过这个值，超出会警告。
    - hard 代表严格阈值，超出就报错。

    ```bash
    ulimit -Sn # 查看警告阈值
    ulimit -Hn # 查看严格阈值

    ulimit -Hn # 设置警告阈值
    ulimit -Sn 10032 # 设置警告阈值
    ```

    通过 ulimit 设置的阈值只在当前会话内有效。想要持久设置，需要修改 /etc/security/limits.conf 文件。

    ```bash
    # /etc/security/limits.conf
    root hard nofile 10032
    root soft nofile 10032
    # root 为进程用户，nofile 代表文件数量。
    ```

    Redis `maxclients` 决定了最大客户端连接，默认 10000。如果同时打开文件数量小于这个数，会建议设置为 maxclient + 32，32 是 Redis 自身需要使用的数量。

    [Redis 客户端文档](https://redis.io/docs/latest/develop/reference/clients/) 中提议设置 `sysctl -w fs.file-max=100000` ，根据[Linux 内核参数文档](https://docs.kernel.org/admin-guide/sysctl/fs.html#file-max-file-nr)，这里设置的是内核最大连接数。可以通过 `cat /proc/sys/fs/file-max` 查看当前值，如果较小，可以增大。

    关于 ulimit 的设置，可以参考 [how-to-set-ulimit-max-number-open-files](https://blog.karmacomputing.co.uk/how-to-set-ulimit-max-number-open-files/)。

- **增大 TCP 全连接队列长度。**

    Redis 默认的 tcp-backlog 值为 511，如果 Linux 系统的全连接队列长度小于 Redis 的值，启动时会警告。Linux 系统相关配置为 `/proc/sys/net/core/somaxconn`，实际采用 max(somaxconn, backlog) 作为应用全连接队列长度。

    ```bash
    # 查看系统默认值
    cat /proc/sys/net/core/somaxconn
    # 查看 6379 端口实际值，Send-Q 列
    ss -lnt | grep 6379
    ```

    设置时将 somaxconn 增加到 Redis tcp-backlog 一致即可。高并发时可以统一为 65536。Redis 需设置 tcp-backlog 65536。

    ```bash
    # 临时设置
    echo 65536 > /proc/sys/net/core/somaxconn
    echo 65536 > /proc/sys/net/ipv4/tcp_max_syn_backlog
    # 持久设置
    echo "net.core.somaxconn=65536" >> sysctl.conf
    echo "net.ipv4.tcp_max_syn_backlog=65536" >> sysctl.conf
    ```

    Docker 在 run 命令添加相关配置。

    ```bash
    docker run ... --sysctl net.core.somaxconn=511 ...
    ```

    Docker compose

    ```yaml
    sysctls:
      net.core.somaxconn: '511'
    ```

### 安全

- **配置网络防火墙。**

    仅限内网访问，禁止外网流量。

    如果是云主机，设置 IP 白名单仅限指定主机访问。

- 使用非 root 用户启动。

    比较简单的做法是用一个非 root 用户安装并配置 Redis。高级一点的做法是专门创建一个用户，nologin 选项禁止登录，并赋予 Redis 相关权限。Docker 默认是以非 root 用户启动。

- （可选）定时备份数据。

    取决于 Redis 数据是否重要。如果是纯粹的缓存服务器，且加载成本不高，可以不备份。

## Redis 服务配置

[Redis 官方的配置文件示例](https://redis.io/docs/latest/operate/oss_and_stack/management/config-file/)

[不同版本的 Redis 配置文件列表](https://redis.io/docs/latest/operate/oss_and_stack/management/config/)

### 调整配置

- **设置最大内存。**

    设置为 70%～80% 物理内存。

    ```bash
    # redis.conf
    # 2GB
    maxmemory 2147483648
    ```

- **开启持久化。**

    对于数据记录场景，数据持久化是基本要求。

    对于缓存场景，开启 Redis 持久化可以加快重启速度。

    建议采取混合持久化方案。

    ```bash
    # redis.conf
    aof-use-rdb-preamble yes
    ```

- 设置客户端连接超时。

    超时客户端连接会被自动释放。使用连接池时没设置 idle 检测，以此兜底。

    ```bash
    # redis.conf
    # 秒
    timeout 300
    ```

- **设置慢日志阈值。**

    放宽慢日志记录条件，增加慢日志队列长度。

    ```bash
    # 命令执行耗时超过 1 毫秒，记录慢日志
    CONFIG SET slowlog-log-slower-than 1000
    # 保留最近 1000 条慢日志
    CONFIG SET slowlog-max-len 1000
    ```

- **限制高危命令防止误操作。**

    部分 Redis 命令的影响很大，比如 `flushdb/flushall` ，误操作会导致数据丢失，需要限制使用。

    高危命令：

    - flushall/flushdb
    - shutdown
    - config
    - debug

    可能导致性能问题的命令：

    - keys
    - save

    Redis 6.0 之前可以通过 `rename-command` 重命名来隐藏命令。

    ```bash
    # 将 flushall 重命名为指定的随机字符串
    rename-command flusall {random_str}
    ```

    重命名命令也有缺陷：

    - 客户端不支持，如果要在客户端使用重命名过的命令，需要修改代码。
    - AOF、RDB 兼容问题。
    - Redis 源码内部调用的命令还是原来的名称，主要是 `config` 命令。
    - 主从复制时需要手动配置从节点，不支持自动设置。

    Redis 6.0 开始，支持 ACL 功能，能更完善地控制权限。ACL 功能支持创建不同用户，并从命令和键名两个维护控制用户权限。
    [ACL 功能文档](https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/)
    [ACL 命令文档](https://redis.io/docs/latest/commands/acl-setuser/)

    ACL 为所有命令添加了不同标签（虽然叫 category 但允许重叠更像是 tag），@dangerous 标签下包含了所有的高危命令，可以通过 `ACL CAT dangerous` 查看。

    ```bash
    # 创建一个叫 application 的用户，密码 42a979...
    # 允许 @dangerous 外的所有命令
    # 支持访问所有的键
    ACL SETUSER application on +@all -@dangerous >42a979... ~*
    ```

- **修改默认端口防止扫描。**

    ```bash
    # redis.conf
    port 9736
    ```

- （可选）调整 hash、zset 等数据类型的 listpack （ziplist）使用条件。

    listpack 是紧凑结构，内存占用比 hashtable 和 skiplist 更低。如果瓶颈在内存，可以提高使用 listpack 的阈值。缺点是命令执行性能可能降低。配置如 `hash-max-listpack-entries` `hash-max-listpack-value`。


### 增加监控

- **监控 bigkey。**

    降低来自 bigkey 的危害。一般是在开发阶段就要注意。

    可以通过 `redis-cli —bigkeys`统计分布。也可以通过 `scan` + `debug object key` 查询实际大小。还可以 Redis RDB Tools 工具，离线分析 rdb 文件。

    **重点是建立自己的 bigkey 标准。**

    可以在从节点上执行分析，降低对业务的影响。

    删除时，通过 unlink 异步删除，对性能影响小，缺点是内存回收不及时。

- **监控热点 Key。**

    可以从客户端角度监控。

    使用代理集群方案时，可以再代理端监控热点。

    可以在 Redis Server 上利用 monitor 命令统计热点。

    在网络层抓包分析。

- 启用延迟监控功能。

    设置一个延迟阈值，超过阈值的事件被记录为延迟峰值。Redis 服务会监控记录命令、fork、rdb、aof、过期、淘汰等多种事件的耗时。

    Redis 官方称延迟监控的实际成本接近于零，可以放心使用。

    ```bash
    # 阈值设置为 100 毫秒
    CONFIG SET latency-monitor-threshold 100
    # 使用 latency 命令查询
    LATENCY latest
    ```

- （可选）监控内存碎片化。

    `used_memory_rss` 代表操作系统分配给 Redis 进程的内存。

    `used_memory` 代表 Redis 分配出去的内存，包括 swap 内存。

    `mem_fragmentation_ratio` 代表内存碎片率，`used_memory_rss` / `used_memory` 。> 1 表示内存碎片严重。< 1 代表存在 swap。

    建议以 used_memory ≥ 500M && mem_fragmentation_ratio > 1.5 作为一个监控指标。

### 追求高并发性能

- **增加最大连接客户端。**

    ```bash
    # redis.conf
    maxclients 20000
    ```

- **增加 tcp-backlog。**

    ```bash
    # redis.conf
    tcp-backlog 65536
    ```

- 读写分离。

    需要在客户端配置。

    ```bash
    # 从节点与主节点断开连接后也能对外提供服务，保证基本可用。
    slave-serve-stale-data yes
    # 从节点仅处理读请求
    slave-read-only yes
    ```

- （可选）增加 I/O 子线程提升网络性能。

    Redis 默认关闭。官方建议确实遇到网络瓶颈再开启，且至少 4 核 CPU 时才开启，注意为主线程预留 1～2 核心。

    ```bash
    # 开启多线程处理接口写入
    io-threads-do-reads yes
    # 配置 I/O 线程数
    io-threads 2
    ```

- （可选）Redis 绑定 CPU 核心，降低上下文切换开销，提高缓存命中率。

    缺点是 fork 子进程会与主进程共用一个 CPU，造成竞争，因此不建议在主节点使用。

    ```bash
    # 在 CPU 0-1 上启用 RPS
    echo '3' > /sys/class/net/eth1/queues/rx-0/rps_cpus
    # redis 的 CPU 亲和性设置为 CPU 2-8
    taskset -pc 2-8 `cat /var/run/redis.pid`
    ```

## 高延迟问题排查

判断是否是 Redis 问题

- [ ]  运行基准测试，得到基准性能数据。与正常实例比较。

确认了是 Redis 性能问题后

- [ ]  检查慢日志，slowlog get {n}，排查慢速命令。应该从命令时间复杂度和数据量两方面分析。
- [ ]  排查 bigkey，redis-cli —bigkeys。
- [ ]  检查持久化方案，AOF + always fsync 很慢。
- [ ]  检查持久化状态，fork 执行时间过长会导致主线程阻塞。info state 查看 latest_fork_usec 指标。
- [ ]  检查内存用量，查看 Redis 进程用量和 swap 情况。
- [ ]  检查系统 THP 是否禁用。
- [ ]  检查 CPU 负载，top 命令查看 Redis 进程 CPU 使用率。
- [ ]  检查服务器网络环境，查看 ulimit fd 数量、backlog 队列长度。
- [ ]  测试网络延迟，在另一台服务器执行 `redis-cli --latency -h host -p port`。
- [ ]  检查 Redis 连接数是否超过 maxclients。

### Redis 基准测试命令

测试固有延迟，必须在 Redis 同台机器上测试。

```bash
# 100s 内的最大响应延迟
redis-cli --intrinsic-latency 100
```

水平有限，欢迎指正。
