# 模块 10：Linux 与生产运维（命令实战 + 线上问题定位）

> **本模块与岗位的关系**
>
> 岗位任职要求：「熟悉 Linux 操作系统原理及常用命令工具（**grep、awk、sed** 等），能在 Linux 环境下**独立完成开发、部署与运维**」。
> 岗位职责第三条更直接：「负责**生产环境线上问题的定位和解决**工作，及时响应并处理系统运行过程中出现的各种问题」
> 「跟进后台服务研发运维工作，与运维团队紧密合作」。
>
> 这意味着这次面试的 Linux 部分**不是背诵题，而是场景题**。面试官的真实心态是：
> 「线上出了事，这个人半夜能不能自己顶住？还是会一直喊运维？」
>
> 所以考察重点有三块，本模块按此展开：
>
> 1. **系统原理**（决定你能不能「推理」出问题原因）：进程/线程/协程、内存管理、文件系统、IO 模型。
> 2. **命令实战**（决定你能不能「动手」找到问题）：grep/awk/sed 的组合场景题是**必考**，几乎一定会让你现场写命令。
> 3. **排查套路**（决定你有没有「体系」）：CPU 高、内存涨、磁盘满、端口占用、网络不通——要能说出**标准流程**，而不是碰运气。
>
> **你的优势素材**：简历里「Docker、Linux」「Flash Root 刷机平台部署」「POS 数据中台」都是真实运维经验。
> 回答时**把命令和你的项目连起来**，例如「我们刷机平台是 Docker 部署的，日志排查我就是这么做的」，
> 比单纯背命令可信十倍。
>
> **复习优先级**：① 命令场景题（必考，能现场写）→ ② 线上排查流程（必问）→ ③ Docker（高频）→ ④ 系统原理（问得深但集中在几个经典题）。

---

## 一、Linux 系统原理

### 1.1 进程、线程、协程

**一句话区分**：进程是**资源分配**的基本单位，线程是**CPU 调度**的基本单位，协程是**用户态**的轻量级调度单位。

| 维度 | 进程 | 线程 | 协程（goroutine） |
|---|---|---|---|
| 资源 | 独立地址空间、文件描述符表、信号处理 | 共享进程的地址空间和资源，独立栈和寄存器 | 共享线程的栈空间，初始栈仅 2KB |
| 切换成本 | 高（切页表、刷新 TLB） | 中（内核态切换，保存寄存器） | **极低**（用户态切换，约几十 ns） |
| 谁调度 | 内核 | 内核 | **Go runtime（用户态调度器）** |
| 数量级 | 几十~几百 | 几百~几千 | **几十万~百万** |
| 通信 | IPC（管道/共享内存/消息队列/信号/socket） | 共享内存（需加锁） | **channel**（CSP 模型，天然同步） |
| 崩溃影响 | 进程隔离，互不影响 | 一个线程崩溃可能拖垮整个进程 | panic 未 recover 会拖垮整个进程 |

**面试深入点：Go 的 goroutine 为什么这么轻？**
> 「三点：**一是栈小且可增长**——初始栈只有 2KB，按需扩缩容，而线程栈通常是 1~8MB 固定；**二是用户态调度**——G、M、P 模型，goroutine 的切换不进入内核，不需要系统调用；**三是阻塞不占线程**——网络 I/O 由 netpoller（底层 epoll）接管，goroutine 阻塞在 I/O 上时 M（OS 线程）会去执行其他 P 上的任务，所以几千个连接不需要几千个线程。」

**协程 vs 线程的实际影响（结合项目讲）**：
> 「我在数字人项目里对接 IM 长连接，如果用线程模型，几千个连接就是几千个线程，内存和切换开销都不可接受；用 goroutine 一个连接一个协程，几 MB 内存就能撑住。这也是 Go 做长连接服务的核心优势。」

**上下文切换的观察命令**：

```bash
vmstat 1            # cs 列 = 每秒上下文切换次数；in 列 = 中断次数
pidstat -w 1        # 按进程看 cswch（自愿，等 I/O）/ nvcswch（非自愿，被抢占）
# 上下文切换过高（十万级）通常意味着：锁竞争激烈、线程/协程过多、或频繁阻塞唤醒
```

### 1.2 进程状态与生命周期

```bash
ps aux              # STAT 列就是进程状态
ps -eo pid,ppid,stat,pcpu,pmem,etime,cmd --sort=-pcpu | head
```

| 状态 | 含义 | 说明 |
|---|---|---|
| **R** (Running/Runnable) | 运行或就绪 | 在 CPU 上执行或在运行队列中等待 |
| **S** (Interruptible Sleep) | 可中断睡眠 | 等待事件（如 socket 数据、锁），**可以被信号唤醒**，最常见 |
| **D** (Uninterruptible Sleep) | 不可中断睡眠 | 通常在等**磁盘 I/O**；**kill -9 也杀不掉**，是磁盘/存储故障的重要信号 |
| **T** (Stopped) | 停止 | 收到 SIGSTOP 或被调试器挂起 |
| **Z** (Zombie) | 僵尸 | 已退出但父进程没回收（没 wait），**不占 CPU/内存，但占 PID 表项** |
| **X** | 已死 | 几乎看不到 |

**面试经典问题**：
1. **「进程 D 状态很多说明什么？」** → 大量不可中断睡眠，通常是**磁盘 I/O 瓶颈或 NFS/存储挂载异常**，用 `iostat -x 1` 看 `%util` 和 `await` 确认。
2. **「僵尸进程怎么处理？」** → 僵尸进程本身杀不掉（已经死了），要**杀掉它的父进程**（父进程退出后会被 init 收养并回收），或者让父进程正确调用 `wait/waitpid`。Go 里如果用 `exec.Command` 后不 `Wait()` 就会产生僵尸。
3. **「孤儿进程呢？」** → 父进程先退出，子进程被 init（PID 1）收养，无害。
4. **「kill -9 和 kill -15 的区别？」** → `-15`（SIGTERM）是**优雅退出**信号，程序可以捕获并做清理（关闭连接、写完日志、注销服务）；`-9`（SIGKILL）是强杀，**不可捕获**，可能导致数据不一致。**生产上先 `-15` 再 `-9`**，K8s 的 `terminationGracePeriodSeconds` 就是这个机制。

```bash
kill -15 <pid>          # 优雅停止（推荐）
sleep 5; kill -9 <pid>  # 5 秒后仍在，再强杀
nohup ./app > app.log 2>&1 &    # 后台运行且不随终端退出
disown -h %1            # 或从作业表中移除
```

### 1.3 内存管理：虚拟内存、页表、缺页中断

**为什么需要虚拟内存**（三个理由，面试答这三点就够）：
1. **隔离**：每个进程有独立地址空间，一个进程崩溃不会破坏别的进程内存。
2. **超售**：进程可以用比物理内存更大的地址空间（靠 swap 和按需分配）。
3. **连续假象**：物理内存是碎片化的，虚拟地址让程序看到连续的地址空间，简化了链接和加载。

**关键机制**：

```
虚拟地址 ──[MMU + 页表]──> 物理地址
                  ↓ 页表项不存在时
              触发缺页中断（page fault）→ 内核分配页框并建立映射 → 重新执行指令
```

- **页（Page）**：通常 4KB，是内存管理的最小单位。**大页（HugePage）** 2MB 可以减少页表项、提高 TLB 命中率（Redis、MySQL 常见调优项）。
- **页表（Page Table）**：多级页表（x86-64 是 4 级），节省空间。
- **TLB**：页表的硬件缓存，**进程切换要刷新 TLB**，所以进程切换比线程切换贵。
- **缺页中断类型**：**主缺页**（minor，页在内存里只是没映射，如写时复制 COW）/ **次缺页**（majflt，需要从磁盘读）——`ps -o min_flt,maj_flt` 能看到。**大量 majflt 说明内存不足在换页，性能会断崖式下降**。
- **写时复制（COW）**：`fork()` 不复制内存，父子共享只读页，谁写谁复制。所以 Redis 的 RDB 持久化 fork 子进程时，内存写入量大就会触发大量 COW 复制，出现内存膨胀。

**内存指标怎么看（生产必用）**：

```bash
free -h
#               total        used        free      shared  buff/cache   available
# Mem:           15Gi        10Gi       1.0Gi       200Mi       4.0Gi        4.5Gi
```

**关键认知（面试高频坑）**：
- **`free` 很小不等于内存不足！** Linux 会用空闲内存做**页缓存（buff/cache）**，这部分**随时可回收**。
- **要看 `available`**（估算的可用内存，包含可回收的 cache），而不是 `free`。
- **`used` 也不完全准**（包含内核和不可回收部分），判断 OOM 风险主要看 `available` 和 `swap` 使用情况。
- `top` 里的 **VIRT / RES / SHR**：VIRT 是虚拟地址空间（包含已申请未使用的，Go 程序 VIRT 通常很大，**不用慌**）；**RES 是实际占用的物理内存，这才是要关注的**；SHR 是共享内存（如动态库）。

```bash
ps -eo pid,rss,vsz,pmem,comm --sort=-rss | head     # 按物理内存排序
smem -r -k | head                                   # 更准的 PSS 统计（共享库按比例分摊）
pmap -x <pid> | tail -1                             # 进程各段内存明细
cat /proc/<pid>/status | grep -E 'VmRSS|VmSize|VmSwap'
slabtop -o | head                                   # 内核 slab 占用（排查内核内存泄漏）
```

### 1.4 文件系统：inode、软硬链接

**核心概念：inode（索引节点）**

```
文件名（存在目录项 dentry 里） ──> inode 编号 ──> inode（元数据）──> 数据块
```

**inode 里存什么**：文件类型、权限、属主、大小、**时间戳（atime/mtime/ctime）**、**指向数据块的指针**、链接计数。
**inode 里不存什么**：**文件名**（文件名在目录项里）、文件内容。

这解释了几个经典现象：
- **同一目录下可以有多硬链接**：多个文件名指向同一个 inode。
- **inode 用尽也会「磁盘满」**：`df -i` 显示 `IUse%` 100%，此时哪怕 `df -h` 还有空间也创建不了新文件（大量小文件场景，如 session 文件、缓存目录）。

```bash
df -h          # 看空间使用率
df -i          # 看 inode 使用率（重要！很多「磁盘满」其实是 inode 满）
```

**软链接 vs 硬链接**：

| | 硬链接（hard link） | 软链接（symbolic link） |
|---|---|---|
| 本质 | 同一个 inode 的**另一个文件名** | 一个**独立文件**，内容是目标路径字符串 |
| inode | **共用**同一个 inode | 有自己的 inode |
| 创建命令 | `ln target link` | `ln -s target link` |
| 能跨文件系统吗 | ❌ 不能（inode 是文件系统内的） | ✅ 能 |
| 能指向目录吗 | ❌ 不能（避免循环） | ✅ 能 |
| 删除原文件 | 文件仍在（只是链接计数 -1） | **链接变成断链**（dangling） |
| `ls -l` 显示 | 只是普通文件（链接计数 > 1） | `link -> target`，且权限是 `lrwxrwxrwx` |

**经典面试题：「为什么删除文件后磁盘空间没释放？」**
> 「最常见的原因是**文件被删除但仍有进程持有它的文件描述符**（比如日志文件被 `rm` 了，但服务进程还开着句柄继续写）。
> 这时目录项没了，但 inode 和数据块要到进程关闭 fd 才释放。用 `lsof | grep deleted` 或者 `ls -l /proc/<pid>/fd | grep deleted` 能找到，
> 处理办法是**重启/重载那个进程**，或者用 `: > /proc/<pid>/fd/<fd>` 清空（把 fd 内容截断），而不是继续 `rm`。」
> —— 这个答案非常实用，我（求职者）在刷机平台遇到过日志目录写满的情况。

**文件描述符（fd）**：

```bash
ulimit -n                      # 当前 shell 的 fd 上限
cat /proc/<pid>/limits | grep 'open files'
lsof -p <pid> | wc -l          # 进程打开的 fd 数量
# 高并发服务的必备调优：ulimit -n 65535 / systemd 里 LimitNOFILE
# 报错 "too many open files" 就是这个限制，或者 fd 泄漏（连接/文件没 Close）
```

### 1.5 五种 IO 模型（网络编程的基础，也是 Go netpoller 的原理）

**先理解两个概念**：一次网络读取分两步——**① 等待数据就绪**（数据到达内核缓冲区）、**② 将数据从内核缓冲区拷贝到用户空间**。五种模型的区别就在于这两步怎么处理。

| 模型 | 等待数据 | 数据拷贝 | 特点 | 典型应用 |
|---|---|---|---|---|
| **阻塞 IO**（Blocking） | 阻塞 | 阻塞 | 一个连接一个线程，简单但并发能力差 | 传统 Java BIO |
| **非阻塞 IO**（Non-blocking） | 轮询不阻塞 | 阻塞 | 需要不断轮询，**空转浪费 CPU** | 少见单独使用 |
| **IO 多路复用** | **阻塞在 select/epoll 上（等多个连接）** | 阻塞 | **一个线程管多个连接**，高并发主流 | Nginx、Redis、**Go netpoller** |
| **信号驱动 IO**（SIGIO） | 不阻塞（信号通知） | 阻塞 | 内核数据就绪后发信号 | 很少用 |
| **异步 IO**（AIO / io_uring） | 不阻塞 | **不阻塞** | 真正异步，全程无阻塞 | io_uring、Windows IOCP |

**面试必答（重点在多路复用）**：

```
阻塞 IO：        read() ──────等待数据───────拷贝──> 返回       （全程占着线程）
多路复用：       epoll_wait() ──等待任一连接就绪──> 再 read()   （一个线程管 N 个连接）
异步 IO：        aio_read() ──立即返回──> 内核全干完──> 通知我   （内核拷贝也帮做了）
```

**select / poll / epoll 的差别（超高频）**：

| 维度 | select | poll | **epoll** |
|---|---|---|---|
| 最大连接数 | **1024**（FD_SETSIZE） | 无硬限制 | 无硬限制 |
| 数据结构 | 位图（fd_set） | 数组（pollfd） | **红黑树 + 就绪链表** |
| 每次调用 | 全量拷贝 fd 集合到内核 + **O(n) 遍历** | 同 select | **拷贝一次（epoll_ctl 注册）**，O(1) 拿就绪 |
| 就绪通知 | 全量返回，用户要自己比对 | 同 select | **只返回就绪的 fd** |
| 触发模式 | 只有 LT | 只有 LT | **支持 LT（默认）和 ET** |
| 适用 | 连接少 | 连接少 | **连接多且活跃比例低**（C10K/C100K） |

**LT 与 ET（面试会追问）**：
- **LT（水平触发，默认）**：只要缓冲区还有数据，每次 `epoll_wait` 都会通知。**安全，不怕漏读**，编程简单。
- **ET（边沿触发）**：只在状态变化时通知一次，**必须一次 read 到 `EAGAIN`**（循环读干净），否则数据会「卡住」不通知。性能略高但容易写错。
- Go 的 netpoller 用的是 **ET 模式 + 循环读**，把 epoll 封装成「goroutine 阻塞读」，让程序员可以按同步方式写代码。

> **话术提示（把 IO 模型接到 Go 上，这是加分回答）**：
> 「Go 的网络模型本质是**多路复用**：runtime 启动时创建 epoll 实例（Linux 下），
> 一个 goroutine 读 socket 时不会阻塞 OS 线程，而是把 fd 注册到 netpoller 上并把自己挂起（状态置为 waiting），
> M（OS 线程）转去执行其他可运行的 goroutine。等 epoll 通知就绪，runtime 唤醒对应的 goroutine 继续执行。
> 所以 Go 可以用**同步的代码写法**获得**异步的性能**——这也是我在数字人项目里一个连接一个 goroutine 就能撑住 IM 长连接的原因。」

**零拷贝（延伸考点，被问到时可以讲）**：

传统文件发送是 4 次拷贝（磁盘→内核缓冲→用户缓冲→socket 缓冲→网卡）；`sendfile` / `splice` 让数据不经用户空间，降到 2~3 次；`mmap + write` 减少一次拷贝。Nginx 静态文件、Kafka 的高吞吐都依赖零拷贝。

---

## 二、常用命令实战（本模块最高频，务必能现场写）

> **先搞清字段位置（面试第一句话就该说的）**
>
> awk 的一切都建立在「第几列是什么」上，所以**动手前先看一行**：
> ```bash
> head -1 access.log        # 或 awk 'NR==1 {for(i=1;i<=NF;i++) print i, $i}'
> ```
>
> 本文档假设 access.log 是 **nginx 默认 combined 格式，并在末尾追加了 `$request_time`（响应耗时，单位秒或毫秒）**：
>
> | 列 | 内容 | 列 | 内容 |
> |---|---|---|---|
> | $1 | 客户端 IP | $8 | 协议版本（HTTP/1.1"） |
> | $4 | 时间 `[17/Sep/2026:14:00:01` | $9 | **状态码** |
> | $6 | 请求方法（"GET） | $10 | 响应体字节数 |
> | $7 | **请求路径 URL** | $11 | **响应耗时（追加字段）** |
>
> **三个实用原则**：
> 1. **不确定列号就先用 `$NF`（最后一列）**——追加字段通常在末尾，用 `$NF` 比写死 `$11` 更稳。
> 2. **面试时主动说一句「我先看一下日志格式」**，比闷头写命令专业得多（也能避免字段错位）。
> 3. 状态码 `$9`、URL `$7`、IP `$1` 是 nginx 格式里**最稳定的三列**，可以放心用。

### 2.1 grep：文本搜索

```bash
# 基础
grep 'error' app.log                      # 包含 error 的行
grep -i 'error' app.log                   # 忽略大小写
grep -v 'DEBUG' app.log                   # 反选（排除）
grep -c 'error' app.log                   # 只输出匹配行数
grep -n 'error' app.log                   # 显示行号
grep -w 'user' app.log                    # 全词匹配（不匹配 username）
grep -o 'ip=[0-9.]*' app.log              # 只输出匹配的部分（提取神器）
grep -r 'TODO' ./src                      # 递归搜索目录
grep -rl 'oldDomain' ./config             # 递归 + 只输出文件名（配合 sed 批量替换）
grep -E 'timeout|refused|reset' app.log   # 扩展正则（或 egrep）
grep -A 3 -B 2 'panic' app.log            # 显示匹配行的后 3 行/前 2 行（-C 3 = 前后各 3 行）
grep --include='*.go' -rn 'TODO' .        # 只在 .go 文件中搜索
```

**常用正则（ERE）速查**：`^` 行首、`$` 行尾、`.` 任意字符、`*` 0+、`+` 1+、`?` 0/1、`[]` 字符集、`|` 或、`()` 分组、`\b` 词边界、`\d`部分版本支持（更稳妥用 `[0-9]`）。

```bash
# 实战：日志里找 500 错误且带 traceID
grep -E '"status":5[0-9]{2}' access.log | grep -o '"traceId":"[^"]*"' | sort -u
```

### 2.2 awk：按列处理（文本分析的瑞士军刀）

**基本结构**：`awk 'pattern { action } END { action }' file`

```bash
# 内置变量
# $0 整行   $1..$n 第 n 列   NF 当前行的列数   NR 行号(全局)   FNR 行号(当前文件)
# FS 输入分隔符（-F 指定）   OFS 输出分隔符   RS 输入行分隔符

# ① 打印指定列
awk '{print $1, $7}' access.log                     # 第 1 列和第 7 列（默认空格分隔）
awk -F: '{print $1}' /etc/passwd                    # 指定冒号分隔 → 用户名
awk -F',' '{print $NF}' data.csv                    # $NF = 最后一列
awk -F'|' '{print $(NF-1)}' data.txt                # 倒数第二列

# ② 条件过滤
awk '$9 == 500 {print $7}' access.log               # 状态码 500 的 URL
awk '$9 >= 400 && $9 < 600 {c++} END {print c}' access.log   # 统计 4xx/5xx 总数
awk 'NR > 1' file.csv                               # 跳过表头
awk 'NR >= 100 && NR <= 200' app.log                # 取 100~200 行
awk 'length($0) > 200' app.log                      # 超长行（排查异常大报文）

# ③ 求和 / 平均 / 最大（按列统计）—— 以「响应耗时」（最后一列 $NF）为例
awk '{sum += $NF} END {print "总耗时:", sum, "平均:", sum/NR}' access.log
awk '{if ($NF > max) {max = $NF; line = $0}} END {print "最慢请求:", max, line}' access.log
awk '{sum[$1] += $NF} END {for (k in sum) print k, sum[k]}' access.log   # 分组求和（按 IP 汇总耗时）

# ④ 格式化输出
awk '{printf "%-15s %s\n", $1, $7}' access.log
awk 'BEGIN {print "IP\tCOUNT"}'                     # BEGIN 块：处理前执行

# ⑤ 关联数组实战：统计每个 URL 的访问量并排序（awk 内部排序不方便，通常交给 sort）
awk '{cnt[$7]++} END {for (u in cnt) print cnt[u], u}' access.log | sort -rn | head -10
```

**awk 常用内置函数**：`length()`、`substr(s,i,n)`、`index(s,t)`、`split(s,arr,sep)`、`gsub(re,rep,s)`、`toupper/tolower`、`int()`、`sprintf()`。

```bash
# 实战：把 URL 中的查询参数去掉再统计
awk '{split($7, a, "?"); cnt[a[1]]++} END {for (u in cnt) print cnt[u], u}' access.log | sort -rn | head
```

### 2.3 sed：流式编辑

**基本结构**：`sed [选项] '地址 命令' file`，**默认不修改原文件，要加 `-i`**。

```bash
# ① 替换
sed 's/old/new/' file            # 每行只替换第一个
sed 's/old/new/g' file           # 全局替换
sed -i 's/old/new/g' file        # 直接修改文件（-i.bak 会先备份）
sed -i 's#/api/v1#/api/v2#g' file    # 用 # 代替 / 作为分隔符，避免转义路径
sed 's/^/PREFIX: /' file         # 行首插入
sed 's/$/;' file                 # 行尾追加分号（CSV 转 SQL 常用）

# ② 按行号/范围操作
sed -n '10,20p' file                     # 打印 10~20 行（-n 抑制默认输出）
sed '10,20d' file                        # 删除 10~20 行
sed -i '3i\新插入的行' file               # 在第 3 行前插入
sed -i '3a\新追加的行' file               # 在第 3 行后追加
sed -i '1,5s/foo/bar/g' file             # 只在 1~5 行替换
sed '$d' file                            # 删除最后一行

# ③ 按内容匹配操作（模式地址）
sed -n '/ERROR/,/END/p' app.log           # 打印从 ERROR 到 END 的区间（含端点）
sed -i '/^$/d' file                       # 删除空行
sed -i '/^#/d' config.conf                # 删除注释行
sed -i '/DEBUG/d' app.log                 # 删除含 DEBUG 的行
sed -i 's/^[[:space:]]*//; s/[[:space:]]*$//' file   # 去首尾空白

# ④ 时间段提取（本项目最实用的招）
sed -n '/2026-09-17 10:00:00/,/2026-09-17 11:00:00/p' app.log > slice.log
```

> **sed 区间的一个真实陷阱（面试说出来很加分）**：`sed -n '/开始/,/结束/p'` 中如果**开始模式一行都没匹配上，输出会是空的**（不是报错）。
> 比如日志里最早的一条是 `14:00:01`，而你写 `/14:00:00/`，结果什么都捞不到，很容易误判成「这段时间没有日志」。
> **稳妥做法**：先用 `grep -c '开始模式'` 确认能匹配上，或者改用 `awk` 的字符串大小比较（不依赖精确匹配）。

**grep vs sed vs awk 的分工（面试常问这个对比，一句话说清）**：
> - **grep**：**查**——按行筛选，回答「哪些行符合条件」。
> - **sed**：**改**——按行编辑（替换/删除/插入），回答「怎么改这些文本」。
> - **awk**：**算**——按列处理、统计、格式化，回答「这些数据算出来是多少」。
>
> 实际工作中三者**组合使用**：grep 粗筛 → awk 提取统计 → sed 输出成想要的样子。

### 2.4 七大高频场景题（面试常让你现场写，务必背下来）

#### 场景 1：统计日志中出现次数最多的 IP（Top 10）

```bash
# 基础版
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# 更健壮的写法（支持逗号/多空格分隔、跳过表头）
awk -F'[ ,]+' 'NR>1 {cnt[$1]++} END {for (ip in cnt) print cnt[ip], ip}' access.log | sort -rn | head -10

# 加一层过滤：只看最近的日志
tail -100000 access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10
```

**为什么这么写？分步解释（面试官爱听）**：`awk` 取第一列 → `sort` 把相同 IP 排到一起（`uniq` 只能处理相邻重复！） → `uniq -c` 计数 → `sort -rn` 按数字倒序 → `head` 取前 10。

#### 场景 2：提取某个时间段的日志

```bash
# 方法一：sed 区间（日志时间有序时最好用，适合大文件）
sed -n '/2026-09-17 14:00:00/,/2026-09-17 15:00:00/p' app.log

# 方法二：awk 字符串比较（更精确，不易误伤）—— 适用于 app.log 行首是「日期 时间」格式，即 $1=日期 $2=时间
awk '$1" "$2 >= "2026-09-17 14:00:00" && $1" "$2 <= "2026-09-17 15:00:00"' app.log
# 如果日志行首只有一个完整时间戳字段（$1 = 2026-09-17T14:00:00.123），直接比较 $1 即可：
awk '$1 >= "2026-09-17T14:00:00" && $1 <= "2026-09-17T15:00:00"' app.log

# 方法三：datetime 格式用字符串前缀过滤就够了（避免解析开销）
grep '^2026-09-17 14:' app.log
grep -E '^2026-09-17 (1[4-5]):' app.log          # 14 点到 15 点

# 组合：提取该时间段的错误日志并统计数量
sed -n '/2026-09-17 14:00:00/,/2026-09-17 15:00:00/p' app.log | grep -c 'ERROR'
```

#### 场景 3：按列统计求和 / 平均值 / 分组

```bash
# 最后一列（$NF）是响应耗时：求总耗时、平均、最大值
awk '{sum+=$NF; if($NF>max) max=$NF} END {print "总:", sum, "平均:", sum/NR, "最大:", max}' access.log

# 按接口分组求平均耗时（第 7 列是 URL，最后一列是耗时）
awk '{sum[$7]+=$NF; cnt[$7]++} END {for (u in sum) printf "%-40s %8.2f ms  (%d 次)\n", u, sum[u]/cnt[u], cnt[u]}' access.log | sort -k2 -rn | head -20

# 统计各状态码分布（$9 在 nginx 格式里固定是状态码）
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# 统计每分钟的请求数（时间格式为 [17/Sep/2026:14:30:01，用 [ ] 做分隔符取中间那段）
awk -F'[][]' '{split($2, t, ":"); print t[2]":"t[3]}' access.log | uniq -c | sort -rn | head

# 补充：如果字段位置不确定，直接按「哪一列像数字/像耗时」来验证
awk 'NR<=3 {for(i=1;i<=NF;i++) printf "%d=%s ", i, $i; print ""}' access.log
```

#### 场景 4：批量替换多个文件中的内容

```bash
# 关键点：grep -rl 先找出「含目标内容的文件」，避免对无关文件做替换
grep -rl 'old-api.company.com' ./config ./src | xargs sed -i 's/old-api\.company\.com/new-api.company.com/g'

# 说明：sed 的 s 命令里 . 需要转义；文件很多时用 xargs -n 1 或 -P 并行
grep -rlZ 'old' . | xargs -0 sed -i 's/old/new/g'

# 只预览不修改（先看要改哪些，这是好习惯）
grep -rl 'old' . | xargs grep -n 'old'

# 目录名批量改（如把 api_v1 改成 api_v2）
find . -type d -name '*v1*' | xargs -I{} sh -c 'mv "{}" "$(echo {} | sed s/v1/v2/)"'
```

#### 场景 5：找出占用空间最大的文件/目录

```bash
# 当前目录下最大的 10 个目录（一层层往下钻，最实用的排查方式）
du -h --max-depth=1 . | sort -rh | head -10
du -sh /var/log/* | sort -rh | head            # 对比多个目标

# 全盘最大的文件（可能慢，注意排除 /proc /sys）
find / -xdev -type f -size +100M -exec ls -lh {} \; 2>/dev/null | sort -k5 -rh | head -20

# 找最大的 10 个文件（用 du 更适合，能正确处理稀疏文件）
du -ah /var 2>/dev/null | sort -rh | head -20

# 删除 7 天前的日志（生产常用）
find /var/log -name '*.log' -mtime +7 -delete
find /var/log -name '*.log' -mtime +7 -exec rm -f {} \;
```

**配套分析**：如果 `du` 和 `df` 结果对不上（`du` 小、`df` 显示满），说明有**已删除但被进程占用的文件**：

```bash
lsof | grep deleted | head
# 或
lsof -nP | grep '(deleted)' | awk '{print $1,$2,$7,$9}'
```

#### 场景 6：统计 TCP 连接状态（排查连接数暴增、TIME_WAIT 堆积）

```bash
# 方法一：ss（推荐，比 netstat 快，现代系统默认有）
ss -ant | awk 'NR>1 {s[$1]++} END {for (k in s) print k, s[k]}' | sort -k2 -rn

# 方法二：netstat（老旧系统）
netstat -ant | awk '{print $6}' | sort | uniq -c | sort -rn

# 只统计 established 数量
ss -ant state established | wc -l
ss -s                                      # 摘要视图，最快

# 按客户端 IP 统计连接数（排查单机连接过多）
ss -ant | awk 'NR>1 {print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head

# 按端口统计（排查哪个服务连接最多）
ss -ant | awk 'NR>1 {print $4}' | awk -F: '{print $NF}' | sort | uniq -c | sort -rn | head
```

**面试延伸（这段说出来很加分）**：
> 「看到 `TIME_WAIT` 很多不用恐慌——它是**主动关闭方**的正常状态，等 2MSL 后自动消失。
> 真正有问题的是：**TIME_WAIT 数量接近本地端口上限**（约 2.8 万）导致无法建立新连接，或者 `CLOSE_WAIT` 大量堆积——
> **`CLOSE_WAIT` 才是真正的 bug 信号**，说明**应用收到了对端的 FIN 但没有调用 close**，通常是代码里连接/响应体没关闭（Go 里就是 `resp.Body` 没 Close 或 `rows.Close()` 忘了）。
> 优化 TIME_WAIT 可以开 `net.ipv4.tcp_tw_reuse=1`（复用，安全），**不要用 `tcp_tw_recycle`**（NAT 环境下会出问题，新内核已删除）。」

#### 场景 7：找出访问量最大的 URL / 慢接口 / 错误分布

```bash
# Top 10 访问 URL
awk '{print $7}' access.log | sort | uniq -c | sort -rn | head -10

# 5xx 错误最多的接口
awk '$9 ~ /^5/ {print $7}' access.log | sort | uniq -c | sort -rn | head -10

# UV（去重后的独立 IP 数）
awk '{print $1}' access.log | sort -u | wc -l
awk '{print $1}' access.log | sort | uniq | wc -l

# 统计每个接口的 QPS 峰值（按秒统计请求数）
awk '{print $4}' access.log | sort | uniq -c | sort -rn | head -5

# 找出耗时超过 1 秒的请求（耗时是最后一列）
awk '$NF > 1000 {print $7, $NF}' access.log | sort -k2 -rn | head -20
```

### 2.5 find 与 xargs

```bash
# find 语法：find [路径] [条件] [动作]
find . -name '*.log'                          # 按名字
find . -iname '*.LOG'                         # 忽略大小写
find . -type f -name '*.go'                   # f 文件 / d 目录 / l 软链接
find . -size +100M                            # 大于 100M（-100k 小于）
find . -mtime -1                              # 1 天内修改过（-mtime +7 = 7 天前）
find . -mmin -30                              # 30 分钟内修改过
find . -perm 644                              # 权限位
find . -user nginx                            # 属主
find . -name '*.tmp' -delete                  # 找到后删除（比 -exec rm 快）
find . -type f -name '*.go' -exec grep -l 'TODO' {} +   # {} + 批量传参（比 \; 快得多）
find . -type f -newer base.txt                # 比某个文件新

# xargs：把标准输入变成命令行参数
cat list.txt | xargs rm -f                    # 批量删除
cat list.txt | xargs -n 1 echo                # 每次传 1 个参数
cat list.txt | xargs -I{} mv {} /backup/{}    # -I 占位符
find . -name '*.log' | xargs -P 4 -n 10 gzip  # -P 4 并行 4 个进程，-n 10 每次传 10 个
find . -print0 | xargs -0 rm                  # -print0/-0 处理含空格的文件名（重要！）
```

**为什么需要 `-print0`/`-0`？** 因为 xargs 默认按空白切分参数，文件名里有空格就会被拆成多个参数（经典事故：删错文件）。用 NUL 字符分隔是唯一安全的做法。

**`find -exec {} \;` 与 `{} +` 的区别**：前者每个文件启动一次命令（慢），后者把所有文件一次传给命令（快）。

### 2.6 sort 与 uniq

```bash
sort file                     # 字典序
sort -n                       # 按数字（否则 10 < 9！）
sort -rn                      # 数字倒序（最常用）
sort -h                       # 人类可读的大小（1K/2M/3G 正确排序）
sort -k2 -rn file             # 按第 2 列数字倒序
sort -t: -k3 -n /etc/passwd   # -t 指定分隔符
sort -u                       # 排序并去重
sort -f                       # 忽略大小写

uniq                          # 只能去掉「相邻」的重复行！必须先用 sort
uniq -c                       # 统计出现次数（最常用）
uniq -d                       # 只显示重复的行
uniq -u                       # 只显示不重复的行
```

**黄金组合（背下来）**：

```bash
sort | uniq -c | sort -rn | head -10     # 「统计 Top N」的万能公式
# 拆解：排序让相同项相邻 → 计数 → 按次数倒序 → 取前 10
```

**`sort -u` 与 `uniq` 的取舍**：仅去重用 `sort -u`（少一次遍历）；**要计数必须用 `sort | uniq -c`**。

### 2.7 进程、端口、网络、磁盘工具速查

```bash
# ---------- 进程 ----------
top -H -p <pid>            # 看进程内各线程的 CPU（Go 程序排查必用）
top -o %CPU                # 按 CPU 排序；按 1 展开每核
ps -ef | grep app          # 查进程
ps -eo pid,ppid,stat,pcpu,pmem,etime,cmd --sort=-pmem | head
pgrep -l -f 'app'          # 按名字/命令找 pid
pstree -p <pid>            # 进程树（看父子关系）
strace -p <pid> -T -tt     # 跟踪系统调用（-T 显示耗时，定位卡在哪个 syscall）

# ---------- 端口 / 连接 ----------
ss -tlnp                   # 监听中的 TCP 端口 + 进程（-p 需要 root 才显示全部）
ss -tnp                    # 已建立的连接
lsof -i:8080               # 谁占用了 8080 端口（最常用）
lsof -i -P -n | grep LISTEN
fuser -n tcp 8080          # 另一种查端口占用的方式
lsof -p <pid>              # 进程打开的所有文件/fd

# ---------- 网络 ----------
ping -c 4 host             # 通不通 + 延迟/丢包
traceroute host            # 路由跳数（哪一跳断了）；mtr 更直观（持续探测）
telnet host 3306           # 端口通不通（没有 telnet 用 nc -vz host 3306）
curl -v -m 5 http://host/health     # 看 HTTP 层细节，-m 5 设 5 秒超时
dig / nslookup host        # DNS 解析
tcpdump -i eth0 -nn port 8080 -w cap.pcap    # 抓包（-nn 不做解析更快）
tcpdump -i any -nn -A 'tcp port 8080 and host 10.0.0.1' | head -50   # 直接看内容
ip a / ip route            # 替代 ifconfig/route
ethtool eth0               # 网卡速率、双工、丢包

# ---------- 磁盘 / IO ----------
df -h / df -i              # 空间 / inode
du -sh * | sort -rh        # 当前目录各项占用
iostat -x 1 5              # 磁盘 IO：%util（繁忙度）、await（平均等待）、r/s w/s
iotop -o                   # 哪个进程在狂读写（-o 只显示有 IO 的）
lsblk / fdisk -l           # 块设备

# ---------- 系统整体 ----------
uptime                     # 负载（1/5/15 分钟），关注「是否超过核数」
vmstat 1                   # r 运行队列 / b 阻塞 / si so 换页 / cs 上下文切换
sar -u 1 5                 # 历史 CPU；sar -n DEV 1 网卡
dmesg -T | tail -50        # 内核日志（OOM、磁盘错误、网卡 down）
journalctl -u app -f --since '10 min ago'   # systemd 服务日志
```

**`uptime` 负载怎么解读（面试常问）**：
> 「`load average` 是**运行队列长度 + 不可中断睡眠进程数**的平均值，包括 1/5/15 分钟。
> 判断标准是**和 CPU 核数比较**：4 核机器负载 4 说明刚好跑满，负载 8 说明有大量任务排队。
> 注意两点：**一是 D 状态（磁盘 I/O 阻塞）也会计入负载**，所以负载高不一定是 CPU 问题，可能是磁盘；
> 二是**要看趋势**——15 分钟负载 10、1 分钟负载 2，说明压力正在下降，反之说明正在恶化。」

---

## 三、线上问题定位套路（岗位职责第三条，本模块的核心得分区）

### 3.1 标准化排查流程（先背这套，面试时按流程讲）

面试官问「线上出问题了你怎么排查」，**不要直接跳到命令**，先给出流程。有流程 = 有方法论 = 靠谱。

```
第 0 步：确认影响面（30 秒内做）
  · 是全部用户还是部分？单机还是集群？哪个接口/功能？
  · 先止损还是先定位？（核心业务挂了 → 先回滚/限流/扩容，再定位根因）
  · 有没有最近上线？→ 有则优先怀疑变更（80% 的故障由变更引起）

第 1 步：看监控大盘（不要一上来就登机器）
  · 应用层：QPS、错误率、P99 延迟、GC、goroutine 数
  · 系统层：CPU、内存、磁盘、网络、连接数
  · 中间件：MySQL 慢查询/连接数、Redis 命中率、MQ 堆积

第 2 步：定位到具体的机器和进程
  · 是单机问题（硬件/部署）还是全集群（代码/依赖）？
  · ssh 上去，先看 top / uptime 建立整体印象

第 3 步：按「资源四要素」逐项排查
  · CPU 高？→ top → top -H → pprof
  · 内存高？→ free → RSS 排序 → pprof heap
  · 磁盘满？→ df -h / df -i → du
  · 网络异常？→ ss / ping / telnet / tcpdump

第 4 步：定位到代码并处理
  · pprof / 日志 / trace 找到具体的函数或 SQL
  · 处理方式：回滚、重启（临时）、限流、修代码后发版

第 5 步：复盘与防护
  · 补监控告警、补压测、补兜底逻辑；写复盘文档
```

**面试话术模板**：
> 「我的习惯是**先止损、再定位、后复盘**。线上第一时间不是找根因，而是先判断影响面和有没有最近变更——
> 如果有变更且影响面大，我会先回滚，把用户影响降到最低，然后再慢慢定位。
> 定位阶段我会按资源维度排查：CPU、内存、磁盘、网络四块，先看监控再登机器，
> 用 `top`/`ss`/`df` 快速缩小范围，再用 pprof 或日志定位到具体代码。」

### 3.2 CPU 飙高排查（最高频的线上问题）

```bash
# ① 整体：找到是哪个进程
top                       # 按 P 按 CPU 排序；看 %us（用户态）、%sy（内核态）、%wa（IO 等待）
uptime                    # 负载与核数对比

# ② 进程内：找到是哪个线程
top -H -p <pid>           # 显示该进程的所有线程（-H），记下高 CPU 的线程 TID
ps -L -p <pid> -o pid,tid,pcpu,comm --sort=-pcpu | head    # 同上，更易读

# ③ 线程 → 代码：Go 程序的做法（推荐，最简单）
# 前提：程序开启了 pprof（import _ "net/http/pprof"）
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
(pprof) top20             # 看 CPU 占用最高的函数
(pprof) list <函数名>      # 看具体哪一行
(pprof) web               # 生成火焰图（需要 graphviz）

# ④ 非 Go 程序：用 perf 或 gdb
perf top -p <pid>                          # 实时看热点函数
perf record -p <pid> -g -- sleep 30 && perf report   # 采样 30 秒后分析
pstack <pid>                               # 打印 C/C++ 进程的调用栈（多打几次找热点）
gdb -p <pid> -batch -ex 'thread apply all bt'   # 看所有线程栈
strace -p <pid> -c -f                      # 统计系统调用（-c 汇总），判断是否卡在 syscall
```

**CPU 高的常见原因（Go 服务）**：

| 原因 | 特征 | 定位方式 |
|---|---|---|
| **死循环 / 忙等待** | 单核跑满 100%，pprof 显示某函数占比极高 | pprof top / list |
| **GC 频繁** | `%us` 高且曲线锯齿状，pprof 有 `runtime.gcBgMarkWorker` | `GODEBUG=gctrace=1`、`go tool pprof` |
| **锁竞争** | 高并发下 CPU 高但 QPS 上不去 | pprof 的 **mutex/block profile**、`go tool pprof .../mutex` |
| **正则回溯（ReDoS）** | 某个接口突然 CPU 飙升 | pprof 显示 `regexp` 相关函数 |
| **JSON 序列化大对象** | pprof 显示 `encoding/json` | 换 easyjson/jsoniter 或减少字段 |
| **日志打太多** | `%sy` 高（write 系统调用） | 检查日志级别、同步写盘 |
| **外部命令/加密计算** | pprof 显示 crypto 或 syscall | 换算法或加缓存 |

> **话术提示（结合项目，非常重要）**：
> 「Go 服务排查 CPU 我有实际经验。我们的做法是**默认在服务里挂 `net/http/pprof` 但只监听内网端口**（比如 `127.0.0.1:6060`），
> 出问题的时候 `go tool pprof` 直接采 30 秒 CPU profile，`top` 一看就知道是哪个函数。
> 我遇到过一次是日志打得太多导致 `%sy` 偏高——因为每条请求都打了完整报文，
> 改成采样打印 + 异步写盘之后 CPU 就降下来了。另一次是缓存没有命中导致大量重复的加解密计算，
> pprof 显示 crypto 函数占了大头，加了一层本地缓存就解决了。」

### 3.3 内存泄漏排查（Go 服务专项）

**先说结论：Go 有 GC，所以「内存一直涨」通常是三类问题**：
1. **goroutine 泄漏**（最常见）——goroutine 挂住不退出，它引用的所有对象都无法回收。
2. **全局缓存/Map 无上限增长**——比如 `map[string]*Conn` 从不删除（逻辑上的内存泄漏）。
3. **切片/子串持有大对象引用**——`big[:2]` 这种（见模块 09 的 1.1）。

```bash
# ① 观察趋势（先确认是不是真的泄漏）
free -h                                    # 整机
ps -o pid,rss,vsz,etime,cmd -p <pid>       # RSS 是否持续增长不回落
cat /proc/<pid>/status | grep -E 'VmRSS|VmSize|VmSwap'
# 连续采样看趋势：
for i in {1..10}; do ps -o rss= -p <pid>; sleep 5; done

# ② Go 服务的 pprof 三件套（核心）
go tool pprof http://localhost:6060/debug/pprof/heap          # 堆内存（inuse_space，当前占用）
go tool pprof -alloc_space http://localhost:6060/debug/pprof/heap   # 累计分配（找分配热点）
go tool pprof http://localhost:6060/debug/pprof/goroutine     # goroutine 数量与堆栈（泄漏必看）
go tool pprof http://localhost:6060/debug/pprof/block         # 阻塞分析（谁在等锁/channel）
go tool pprof http://localhost:6060/debug/pprof/mutex         # 锁竞争

# pprof 常用交互命令
#   top10            按占用排序
#   list FuncName    看函数内每行的内存分配
#   web              图形化
#   peek / traces    看调用链

# ③ 兜底：如果内存涨到快 OOM，先保留现场再重启
curl -s http://localhost:6060/debug/pprof/heap > /tmp/heap.$(date +%s).pb.gz
curl -s http://localhost:6060/debug/pprof/goroutine?debug=2 > /tmp/goroutine.txt   # debug=2 带完整堆栈
```

**goroutine 泄漏的四个典型原因（Go 面试超高频，见模块 02 也可呼应）**：

| 原因 | 例子 | 修法 |
|---|---|---|
| channel 收发无人配对 | `ch <- v` 但没人接收 | 发送/接收都要有退出路径（select + ctx.Done） |
| 忘记取消 context | 派生 ctx 后没有 `defer cancel()` | 一律 `defer cancel()` |
| 死锁/等待锁 | 持锁时做慢操作 | 缩小锁范围、加超时 |
| 后台循环没有退出条件 | `for { ... }` 没有 ctx 检查 | 用 ctx 控制生命周期 |

```go
// ❌ 泄漏示例
func leak() {
    ch := make(chan int)
    go func() { ch <- 1 }()      // 这个 goroutine 永久阻塞在发送上
    // 没有接收方 → goroutine 泄漏
}

// ✅ 修法
func ok(ctx context.Context) {
    ch := make(chan int, 1)      // ① 缓冲 1，发送不阻塞
    go func() { ch <- 1 }()
    select {
    case v := <-ch:
        _ = v
    case <-ctx.Done():
        return                   // ② 有退出路径
    }
}
```

**线上判断是否泄漏的辅助工具**：

```bash
# 用 GODEBUG 看 GC 情况（日志里输出每次 GC 的详细数据）
GODEBUG=gctrace=1 ./app
# 关注：gc N @Xs X%: ... 的堆大小是否持续增长、GC 频率是否变密

# 用 expvar 或 Prometheus 暴露关键指标（生产必做）
#   go_goroutines          当前 goroutine 数（持续上涨 = 泄漏）
#   go_memstats_heap_inuse_bytes   堆占用
#   go_gc_duration_seconds GC 耗时
```

### 3.4 磁盘满排查

```bash
df -h                       # ① 哪个分区满
df -i                       # ② 是不是 inode 满（小文件多）

du -h --max-depth=1 / | sort -rh | head    # ③ 逐层定位大目录
du -sh /var/log/* /tmp/* /home/* 2>/dev/null | sort -rh | head

# ④ 定位大文件
find / -xdev -type f -size +500M -exec ls -lh {} \; 2>/dev/null

# ⑤ 已删除但未释放的文件（du 和 df 对不上的元凶）
lsof -nP | grep '(deleted)' | head
# 处理：重启持有该文件的进程，或清空 fd（下面这条会截断文件）
: > /proc/<pid>/fd/<fd>

# ⑥ 常见「空间杀手」清单
#   /var/log（日志没轮转）→ 用 logrotate 或程序内切割
#   Docker 的 /var/lib/docker → docker system prune -a
#   被 journald 占满 → journalctl --vacuum-size=500M
#   内核转储 /var/crash → 清理
```

**预防措施（面试官爱听这话）**：
> 「根治办法不是出事了再删，而是**日志轮转（logrotate 或 lumberjack）+ 磁盘水位告警（80% 报警）+ 关键服务单独挂数据盘**。
> 我们的服务统一用 lumberjack 做日志切割，保留 7 天，并在监控里配了磁盘使用率告警。」

### 3.5 端口占用排查

```bash
# 谁占用了 8080
lsof -i:8080
ss -tlnp | grep 8080
netstat -tlnp | grep 8080
fuser -n tcp 8080                 # 直接输出 pid

# 处理
kill -15 <pid>                    # 优雅停止
# 如果 kill 不掉（D 状态或僵尸），检查父进程 / 是否有僵死容器

# 检查端口是否被监听（服务有没有起来）
ss -tlnp | grep -E ':(8080|6060)'
# 注意：监听 127.0.0.1 和 0.0.0.0 的区别！
#   127.0.0.1:8080 → 只能本机访问（对外不通，常见配置错误）
#   0.0.0.0:8080   → 所有网卡可访问
#   容器里若监听 127.0.0.1，端口映射出来也访问不到（经典坑）
```

### 3.6 网络不通排查（分层排查，思路清晰最重要）

```
第 1 层：本机网络是否正常
  ip a                       # 网卡是否有 IP、是否 up
  ping 127.0.0.1             # 协议栈是否正常
  ping 网关                   # 是否通内网

第 2 层：DNS / 域名解析
  nslookup api.company.com
  dig api.company.com +short
  cat /etc/resolv.conf
  # 常见问题：DNS 解析失败 / 解析到旧 IP / /etc/hosts 覆盖

第 3 层：路由是否可达
  ping <目标IP>              # ICMP 通不通（注意：有些机器禁 ICMP，ping 不通不代表服务不通）
  traceroute <目标IP>        # 在哪一跳断了（mtr 更直观，可连续观察丢包）
  ip route get <目标IP>

第 4 层：端口是否可达（最关键的一步）
  telnet <ip> <port>         # 或 nc -vz <ip> <port>
  # 报 Connection refused → 端口没监听（服务没起/监听地址不对/防火墙 reject）
  # 卡住无响应（超时）     → 被防火墙 DROP / 网络不通 / 安全组未开

第 5 层：应用层
  curl -v -m 5 http://<ip>:<port>/health     # HTTP 层是否正常
  # 看是 TCP 连不上，还是连上了但返回 4xx/5xx

第 6 层：抓包（前面都正常但业务不通时）
  tcpdump -i any -nn port <port> -w /tmp/cap.pcap
  # 分析：有没有收到 SYN？有没有回 SYN-ACK？有没有 RST？只有单向包？
  #  只有 SYN 无回应 → 对端没收到或被防火墙丢弃
  #  收到 RST        → 端口未监听，或被中间设备重置
  #  三次握手完成但没数据 → 应用层问题
```

**面试话术（分层思路比命令更重要）**：
> 「网络问题我会**分层排查**：先本机（`ip a`、ping 网关），再 DNS，再路由（`traceroute`），
> 再端口（`telnet` 最直接），最后应用层（`curl -v`）和抓包（`tcpdump`）。
> 关键经验是**区分 `Connection refused` 和超时**——refused 说明网络是通的、只是端口没监听（服务挂了或者监听在 127.0.0.1）；
> 超时才是网络或防火墙问题。这个区分能省下大量排查时间。」

### 3.7 OOM 与 dmesg

**先分清两种 OOM**：
1. **系统 OOM Killer**：整机内存耗尽，内核杀进程（选择 oom_score 最高的，通常是内存占用最大的）。
2. **容器 OOM**：cgroup 内存限制被突破，容器被杀（K8s 里表现为 `OOMKilled`，Pod 重启）。

```bash
# 确认是否发生过 OOM
dmesg -T | grep -i -E 'oom|killed process'
dmesg -T | grep -i 'out of memory'
journalctl -k | grep -i oom                     # systemd 系统
# 典型输出：
#   Out of memory: Killed process 12345 (app) total-vm:8000000kB, anon-rss:3500000kB

# 看 OOM 评分（分数越高越先被杀）
cat /proc/<pid>/oom_score
cat /proc/<pid>/oom_score_adj                  # -1000 表示永不被杀（谨慎使用）

# 容器场景
docker inspect <container> | grep -i oomkilled
kubectl describe pod <pod> | grep -A5 'Last State'   # 看 Reason: OOMKilled
cat /sys/fs/cgroup/memory/memory.limit_in_bytes      # cgroup v1 内存上限
cat /sys/fs/cgroup/memory.max                        # cgroup v2

# 内核参数（了解即可，生产慎改）
vm.overcommit_memory        # 0 启发式 / 1 总是允许 / 2 不允许超售
vm.swappiness               # 换页倾向，服务器常用 0~10
vm.panic_on_oom             # OOM 时是否 panic
```

**Go 程序被 OOM 杀的特殊性（加分内容）**：
> 「Go 程序被 OOM 杀掉时**看不到 panic 堆栈**，因为 SIGKILL 无法捕获。
> 所以排查 Go 的 OOM 要靠两样东西：**一是提前抓 heap profile**（用定时任务定期 dump，或者内存超过阈值自动 dump）；
> **二是 `GOMEMLIMIT`**（Go 1.19+）——它让 GC 知道内存上限，会在接近限制时更激进地回收，
> 在容器里配合 cgroup 限制使用，可以显著减少被 OOM Killer 杀掉的概率。另外 `GOGC` 调小可以让 GC 更积极（代价是 CPU）。」

### 3.8 其他高频线上问题速查表

| 现象 | 第一反应命令 | 常见原因 |
|---|---|---|
| 接口变慢 | 看 P99 监控 → `top` → MySQL 慢查询日志 | 慢 SQL、连接池耗尽、下游超时、GC 停顿 |
| 请求超时 | `ss -s`、`netstat -s` | 连接数打满、TIME_WAIT/CLOSE_WAIT 堆积、半连接队列溢出 |
| 服务假死（不响应但进程在） | `go tool pprof .../goroutine?debug=2` | goroutine 全阻塞（死锁、等锁、channel 满） |
| 日志暴涨 | `du -sh /var/log/*` | 循环里打日志、异常被大量重试打印 |
| 定时任务重复执行 | 看部署脚本/多实例 | 多副本部署没加分布式锁 |
| 文件句柄耗尽 | `lsof -p <pid> \| wc -l`、`ulimit -n` | fd 泄漏（没 Close）、上限太低 |
| 时钟漂移导致异常 | `date`、`ntpq -p` | NTP 未同步，影响 JWT/签名/日志时序 |
| 磁盘 IO 高 | `iostat -x 1`、`iotop -o` | 大文件读写、日志同步写、swap 换页 |

### 3.9 把项目经验接上去（面试话术）

> **刷机平台（Docker 部署）**：
> 「Flash Root 刷机平台是我们自己 Docker 部署的，我用得最多的就是 `docker logs` + `docker stats` 看容器状态，
> 出问题就 `docker exec` 进去看进程和文件。有一次是磁盘写满导致上传失败——用 `df -h` 定位到分区，
> 再用 `du -sh` 一层层找到是日志目录，加上 logrotate 之后就没再犯过。」
>
> **数据中台（Go + MQ 消费）**：
> 「数据中台的服务是 Go 写的，我给它挂了 pprof 内网端口。有一次消费延迟变大，
> 我从监控看到 goroutine 数一直涨，用 `pprof goroutine` 一抓就发现是某个 RabbitMQ 消费协程在异常路径下没有退出，
> 卡在无缓冲 channel 的发送上。修法是加 ctx 取消路径 + 给 channel 加缓冲。这个经历让我现在写并发代码一定会检查『有没有退出路径』。」
>
> **商云前台（客户端）**：
> 「桌面端我们也有日志目录和本地 SQLite，遇到过打印任务堆积导致本地文件暴涨的情况，
> 后来加了队列长度上限和落盘清理策略。」

---

## 四、Docker（岗位明确要求，简历里也写了）

### 4.1 镜像分层与联合文件系统

**核心机制**：
- 镜像由**多个只读层（layer）**叠加而成，每一条 Dockerfile 指令（`RUN`/`COPY`/`ADD`）产生一层。
- 容器启动时在镜像顶部加一个**可写层（container layer）**，所有修改都写在可写层——**这就是「容器删了数据就没了」的原因**，需要持久化必须用**数据卷（volume）**。
- 底层用 **overlay2**（联合文件系统，UnionFS）把多层合并成一个统一的文件系统视图。
- **分层带来两个好处**：① **层可以复用**（多个镜像共享基础层，节省磁盘）；② **构建缓存**（某一层没变，后续层直接复用缓存）。

```
容器可写层  ← 容器运行时的修改（删除后消失）
─────────────
layer N     ← COPY app                  (变了则这层及以下都重建)
layer N-1   ← RUN go build              (依赖源码)
layer 2     ← RUN apk add ca-certificates
layer 1     ← FROM alpine:3.19          (基础镜像，多镜像共享)
```

```bash
docker history <image>            # 看镜像每一层的大小和来源命令（排查镜像为什么这么大）
docker image inspect <image> | grep -A20 RootFS
docker system df                  # 镜像/容器/卷占用的空间
docker system prune -a            # 清理未使用的镜像和容器（磁盘救急）
```

**面试经典题：「为什么 Dockerfile 里 `COPY . .` 要放在 `go mod download` 之后？」**
> 「因为**层缓存是按顺序命中的**：一旦某一层的内容变了，它和它后面的所有层都要重建。
> 如果先 `COPY . .` 再 `go mod download`，那么每次改一行代码都会导致依赖重新下载（几分钟）；
> 反过来先 COPY `go.mod`/`go.sum` 再 `RUN go mod download`，代码改动不会让依赖层失效，构建从几分钟降到几秒。
> 这是 Dockerfile 优化里收益最大的一条。」

### 4.2 Dockerfile 优化（Go 服务实战模板）

```dockerfile
# ============ 阶段一：构建 ============
FROM golang:1.22-alpine AS builder

WORKDIR /build

# ① 先只拷贝依赖清单 → 依赖层可复用（最重要的一条）
COPY go.mod go.sum ./
RUN go env -w GOPROXY=https://goproxy.cn,direct && \
    go mod download

# ② 再拷贝源码 → 代码变更只影响后面的层
COPY . .

# ③ 静态编译：CGO_ENABLED=0 不依赖 glibc，才能跑在 scratch/alpine 上
#    -s -w 去掉符号表和调试信息，产物可减小 20~30%
#    -trimpath 去掉构建机器的路径信息（也更安全）
RUN CGO_ENABLED=0 GOOS=linux go build \
    -trimpath \
    -ldflags="-s -w -X main.version=$(date +%Y%m%d%H%M)" \
    -o /build/app ./cmd/server

# ============ 阶段二：运行（多阶段构建，最终镜像不含 Go 工具链）============
FROM alpine:3.19

# 时区 + 证书（调用 HTTPS 必须要证书）—— 这两条是 Go 镜像最常见的坑
RUN apk add --no-cache ca-certificates tzdata && \
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone

# 非 root 用户运行（安全基线要求）
RUN adduser -D -u 10001 appuser

WORKDIR /app
COPY --from=builder /build/app /app/app
COPY --from=builder /build/configs /app/configs

USER appuser
EXPOSE 8080

# HEALTHCHECK 让编排系统知道容器是否健康
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD wget -qO- http://127.0.0.1:8080/health || exit 1

# 用 ENTRYPOINT（exec 形式）+ CMD 提供默认参数
ENTRYPOINT ["/app/app"]
CMD ["-conf", "/app/configs/config.yaml"]
```

**优化要点汇总（面试直接按这个讲）**：

| 优化点 | 做法 | 收益 |
|---|---|---|
| **多阶段构建** | builder 阶段编译，运行阶段只拷贝二进制 | 镜像从 ~800MB 降到 ~20MB |
| **基础镜像选择** | alpine / distroless / scratch | 体积小、攻击面小 |
| **层缓存** | 依赖清单先 COPY，源码后 COPY | 构建从分钟级到秒级 |
| **静态编译** | `CGO_ENABLED=0` | 可以跑在 scratch 上 |
| **剪裁二进制** | `-ldflags="-s -w"`、`-trimpath` | 体积 -20%~30%，且不泄露路径 |
| **.dockerignore** | 排除 `.git`、`node_modules`、日志 | 减小构建上下文，加快构建 |
| **非 root 运行** | `USER appuser` | 安全基线 |
| **时区与证书** | 装 `tzdata` + `ca-certificates` | 避免时间错乱和 HTTPS 失败 |
| **依赖代理** | `GOPROXY` 指向国内源 | 构建稳定 |

```bash
# .dockerignore 示例（必须写，否则 COPY . . 会把 .git 和 node_modules 都塞进构建上下文）
.git
.gitignore
*.md
node_modules
dist
logs
*.log
.env
tmp
```

**`CMD` vs `ENTRYPOINT`（必考）**：

| | `ENTRYPOINT` | `CMD` |
|---|---|---|
| 语义 | 容器的**主命令**（固定不变） | **默认参数**（可被覆盖） |
| `docker run image arg` | arg 作为参数追加给 ENTRYPOINT | **完全替换** CMD |
| 形式 | 建议 exec 形式 `["app"]` | 建议 exec 形式（shell 形式会让进程变成 PID 1 的 shell 子进程，**收不到 SIGTERM，无法优雅退出**） |

**面试加分点：「为什么推荐 exec 形式？」**
> 「shell 形式（`CMD app -c cfg`）会被包装成 `/bin/sh -c "app -c cfg"`，那么 **PID 1 是 sh，不是你的程序**。
> 结果是 `docker stop` 发的 SIGTERM 打给了 sh，你的 Go 程序收不到信号，无法优雅关闭（关闭连接、写完数据），
> 10 秒后被 SIGKILL 强杀，可能丢数据。exec 形式让程序成为 PID 1，能正常接收信号、正确退出码。
> 另外 Go 程序作为 PID 1 时要注意**回收僵尸子进程**（如果会 fork 子进程），可以用 `tini` 作为 init。」

### 4.3 容器网络模式

| 模式 | 说明 | 使用场景 |
|---|---|---|
| **bridge**（默认） | 容器接入 docker0 虚拟网桥，独立 namespace，通过 NAT 出网；容器间用容器名/IP 互访 | 单机多容器（用自定义 bridge，才有 DNS 名称解析） |
| **host** | 与宿主机共享网络 namespace（**没有独立 IP，端口直接占用宿主机的**） | 高性能场景（省去 NAT 开销）、需要拿到真实客户端 IP |
| **none** | 只有 lo，无网络 | 完全隔离的任务（离线计算） |
| **container:<name>** | 共享另一个容器的网络 namespace | Sidecar 模式（如网络代理） |
| **overlay** | 跨主机的虚拟网络（Swarm/K8s 底层用类似机制） | 多机集群通信 |

```bash
# 自定义 bridge 网络（推荐，容器间可用名字互访 + 隔离性好）
docker network create mynet
docker run -d --name db --network mynet mysql:8
docker run -d --name app --network mynet -e DB_HOST=db myapp    # 直接用容器名当主机名

# 端口映射
docker run -p 8080:8080 myapp            # 宿主机8080 → 容器8080
docker run -p 127.0.0.1:8080:8080 myapp  # 只绑本机（安全，防止公网直接访问）
docker run -P myapp                      # 随机映射所有 EXPOSE 端口

# 排查容器网络
docker exec -it app sh                   # 进容器
docker inspect -f '{{.NetworkSettings.IPAddress}}' app
docker port app
# 经典坑：容器内服务监听 127.0.0.1 → 端口映射无效，必须监听 0.0.0.0
```

**为什么容器里的服务必须监听 `0.0.0.0`？** 因为 `-p 8080:8080` 是通过 **DNAT** 把宿主机流量转发到**容器 IP**的，如果进程只监听容器内的 127.0.0.1，转发过来的目标地址（容器 IP）就没人接——**这是新手最常踩的坑之一**。

### 4.4 常用命令速查

```bash
# ---------- 镜像 ----------
docker build -t registry.company.com/app:v1.0.0 .
docker build --no-cache -t app:test .          # 不用缓存（排查构建问题时）
docker images                                   # 列出镜像
docker pull / push registry/app:v1
docker tag app:v1 registry/app:v1
docker rmi <image>                              # 删镜像
docker history app:v1                           # 看每层（排查体积）

# ---------- 容器 ----------
docker run -d --name app -p 8080:8080 \
  -v /data/app/logs:/app/logs \
  -e ENV=prod --restart=always \
  --memory=1g --cpus=1.5 \
  app:v1
docker ps -a                                    # 含已退出
docker logs -f --tail 200 app                   # 跟日志（最常用）
docker exec -it app sh                          # 进容器
docker stats                                    # 实时资源占用
docker inspect app | jq '.[0].State'            # 容器状态详情
docker restart / stop / start app
docker rm -f app                                # 强制删除
docker cp app:/app/log.txt ./                   # 拷文件出来
docker top app                                  # 容器内进程（宿主机视角）

# ---------- 数据卷 ----------
docker volume create appdata
docker volume ls / inspect appdata
docker run -v appdata:/app/data app:v1          # 命名卷（推荐，Docker 管理）
docker run -v /host/path:/app/data app:v1       # 绑定挂载（宿主机路径，注意权限）

# ---------- 编排（单机多容器）----------
docker compose up -d / down / logs -f / ps / restart app / exec app sh

# ---------- 清理（磁盘救急）----------
docker system df                                # 看占用构成
docker system prune -a --volumes                # 危险！清理所有未使用的镜像/容器/卷（生产慎用）
```

**面试常问「数据卷和绑定挂载的区别」**：
> 「命名卷由 Docker 管理，存在 `/var/lib/docker/volumes` 下，跨平台一致、可以备份迁移、权限由 Docker 处理，**是生产推荐做法**；
> 绑定挂载直接映射宿主机路径，方便但依赖宿主机目录结构和权限（容器内用户 uid 和宿主机不一致就会 Permission denied），
> 适合挂配置文件、日志目录这种需要人直接看的场景。」

### 4.5 Kubernetes 简述（了解层次即可，问到时能聊天）

**和 Docker 的关系**：Docker 是**单机容器运行时**（打包 + 运行）；K8s 是**容器编排系统**（调度到哪台机器、扩缩容、自愈、服务发现、滚动发布）。现在 K8s 通过 CRI 调用 containerd（Docker 的运行时被拆了出去，所以「K8s 弃用 Docker」指的是不再用 dockershim，镜像格式仍然兼容 OCI）。

**核心对象（能说清 5 个就够）**：

| 对象 | 作用 | 一句话 |
|---|---|---|
| **Pod** | 最小调度单位 | 一个或多个共享网络/存储的容器；**Pod 内的容器共享 localhost** |
| **Deployment** | 无状态应用部署 | 管理 ReplicaSet，负责副本数、滚动更新、回滚 |
| **Service** | 稳定访问入口 | 为一组 Pod 提供固定 ClusterIP + 负载均衡（Pod IP 会变，Service 不变） |
| **Ingress** | 七层入口 | 按域名/路径转发到不同 Service（替代一堆 NodePort） |
| **ConfigMap / Secret** | 配置与密钥 | 环境变量或挂载文件注入，Secret 是 base64（**不是加密**，需要额外加密方案） |
| **StatefulSet** | 有状态应用 | 稳定的网络标识 + 独立存储（MySQL、Kafka） |
| **HPA** | 自动扩缩容 | 按 CPU/QPS 指标自动调整副本数 |

```bash
# 日常运维命令（面试能说出这些就很够用）
kubectl get pods -n prod -o wide
kubectl describe pod app-xxx -n prod          # 排查看 Events（最重要的排查手段）
kubectl logs -f app-xxx -n prod --tail=200
kubectl exec -it app-xxx -n prod -- sh
kubectl rollout restart deployment/app -n prod
kubectl rollout status / undo deployment/app  # 查看/回滚发布
kubectl top pods -n prod                      # 资源占用
```

**面试被问「你们怎么部署的」怎么答（结合项目）**：
> 「Flash Root 刷机平台我们是**单机 Docker + docker compose** 部署的——服务不多，用 compose 编排 MySQL、Redis 和 Go 服务足够，
> 加上 `restart: always` 和健康检查，稳定性够用。公司有专门的运维团队管 K8s 集群，
> 我会用 `kubectl` 看 Pod 状态和日志、拉起来排查，但集群本身的维护不是我负责的。
> 我对 K8s 的理解是：**它解决的是「多机 + 多副本」下的调度、自愈和发布问题**——Pod 挂了自动重建、滚动发布、
> 通过 Service 做服务发现和负载均衡。我们后端服务上云之后就是这个模式。」

---

## 五、高频面试题与参考答案（12 题）

**Q1：`ps` 里进程的 STAT 都有哪些状态？D 状态多说明什么？**
> 见 1.2。要点：R 运行/就绪、S 可中断睡眠（最常见）、D **不可中断睡眠（等磁盘 I/O，kill -9 也杀不掉）**、T 停止、Z 僵尸。
> D 状态多说明**磁盘 I/O 瓶颈或存储挂载异常**，用 `iostat -x 1` 看 `%util`/`await` 确认。
> 补充：`Z` 僵尸进程本身杀不掉，要处理它的父进程。

**Q2：硬链接和软链接的区别？**
> 见 1.4 表格。核心三点：**硬链接共用 inode、不能跨文件系统、不能指向目录**；软链接是独立文件存路径、能跨文件系统能指向目录、**原文件删了就断链**。
> 加分：`inode` 用尽也会「磁盘满」（`df -i`），大量小文件场景要注意。

**Q3：五种 IO 模型说一下，epoll 比 select 好在哪？**
> 见 1.5。先说两阶段（等数据 + 拷贝数据），再说五种模型的差别。
> select：**1024 上限、每次全量拷贝、O(n) 遍历**；epoll：**红黑树注册一次、就绪链表只返回有事件的 fd、O(1)**。
> LT/ET 的区别要能说清（ET 必须读到 EAGAIN）。
> 收尾接到 Go：「Go 的 netpoller 就是 epoll 的封装，所以在 Go 里一个连接一个 goroutine 也不会崩。」

**Q4：统计日志中出现次数最多的 IP，怎么写？**
> 见 2.4 场景 1。**必须能现场写出来**：
> `awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10`
> 并解释为什么要先 `sort`（`uniq` 只能处理相邻重复行）。
> 追问「大文件怎么办」：先 `tail -n 1000000` 或者用 `split` 分片 + 分别统计再合并（MapReduce 思路）。

**Q5：如何在 10GB 日志里快速找出某个时间段且包含 ERROR 的记录？**
> `sed -n '/起始时间/,/结束时间/p' app.log | grep ERROR`
> 说明取舍：「如果日志按时间有序，`sed` 区间最快；如果不确定有序，用 `awk` 做字符串比较更稳（`$1" "$2 >= "..."`）。
> 再大就用 `grep '^2026-09-17 14:'` 这种前缀匹配，直接走字符串扫描，不用正则回溯。」

**Q6：怎么找出磁盘空间去哪了？如果 `df` 显示满但 `du` 加起来没那么多呢？**
> 见 3.4。分两层：① 正常定位用 `df -h` → `du -h --max-depth=1` 逐层下钻 → `find` 找大文件；
> ② **`df` 满但 `du` 对不上** = 有**已删除但被进程占用的文件**，用 `lsof | grep deleted` 找，
> 处理方式是重启进程或 `: > /proc/<pid>/fd/<fd>` 清空。
> 补一句预防：「日志轮转 + 磁盘水位告警才是根治。」
> **话术提示**：可结合刷机平台磁盘写满的亲身经历。

**Q7：线上 CPU 100% 你怎么排查？**
> 见 3.2 的完整链路：`top` 找进程 → `top -H -p` 找线程 → Go 用 `pprof` / 其他用 `perf`、`pstack` 找函数 → 定位代码。
> 再说常见原因（死循环、GC 频繁、锁竞争、正则回溯、日志过多）。
> **一定要说 pprof**（Go 岗位的加分点），并说明「我们只监听内网端口，安全」。
> 补充：「如果 SYS 高（%sy），通常是系统调用频繁——比如日志同步写盘、频繁的小 IO；如果 %wa 高，那是 IO 等待不是 CPU 问题，要查磁盘。」

**Q8：Go 服务内存一直涨怎么办？**
> 见 3.3。先说三类原因（goroutine 泄漏、全局缓存无上限、切片持有大对象），
> 再说工具（`pprof heap` 看 inuse_space、`pprof goroutine?debug=2` 看泄漏堆栈），
> 最后讲防（监控 `go_goroutines` 指标、`GOMEMLIMIT`/`GOGC` 调优、代码上「凡 go 必想退出路径」）。
> **话术提示**：结合数据中台 RabbitMQ 消费 goroutine 泄漏的真实经历讲，非常有说服力。

**Q9：`kill -9` 和 `kill -15` 的区别？容器里 `docker stop` 发生了什么？**
> `-15`（SIGTERM）可捕获，程序做优雅退出；`-9`（SIGKILL）不可捕获，直接杀。
> `docker stop` 先发 SIGTERM，等待 10 秒（默认）后发 SIGKILL。
> 加分：**所以 Go 服务必须处理 SIGTERM 信号**做优雅关闭（停止接收新请求、处理完存量请求、关闭连接池），否则每次发布都会掉请求。
> 再补一句 Dockerfile 的坑：**shell 形式的 CMD 会让 sh 成为 PID 1，SIGTERM 传不到程序**，所以要用 exec 形式。

**Q10：Docker 镜像为什么这么大？怎么优化？**
> 见 4.2 表格。按收益排序讲：**多阶段构建**（最大收益）→ **基础镜像换 alpine/distroless** → **层缓存顺序**（依赖先 COPY）
> → **`-ldflags="-s -w"` + `-trimpath`** → **.dockerignore**。
> 举例：「我们的 Go 服务从 800MB 优化到 20MB 左右。`docker history` 可以看每一层的来源和大小，是排查体积的第一步。」

**Q11：容器网络有哪几种模式？为什么容器里的服务要监听 0.0.0.0？**
> 见 4.3。四种模式（bridge/host/none/container）+ overlay。
> 0.0.0.0 的问题：**端口映射本质是 DNAT 到容器 IP**，如果只监听 127.0.0.1 就收不到转发流量——**这是最高频的部署坑**。

**Q12：如何查看某个端口被哪个进程占用？如何确认服务是否正常监听？**
> `lsof -i:8080` 或 `ss -tlnp | grep 8080`；处理用 `kill -15`。
> 加分：**区分 `127.0.0.1:8080` 和 `0.0.0.0:8080`**——前者只有本机能访问，
> 「服务起来了但外面访问不了」十有八九是这个原因，也直接影响容器端口映射。

---

## 六、自测清单（命令必须能默写）

1. 默写：统计 access.log 中出现次数最多的 10 个 IP（并解释为什么要先 sort）。
2. 默写：提取 14:00~15:00 之间且包含 ERROR 的日志行。
3. 默写：统计第 10 列（耗时）的总和、平均、最大值；并按 URL 分组求平均。
4. 默写：把当前目录所有 `config/*.yaml` 里的 `old.host` 批量替换为 `new.host`（先预览再改）。
5. 默写：找出 `/var` 下最大的 10 个目录和最大的 10 个文件（再写一条找 7 天前日志并删除）。
6. 默写：统计本机 TCP 各状态连接数；按客户端 IP 统计连接数 Top10。
7. 口述：CPU 飙高的完整排查链路（top → top -H → pprof → 代码），并说出 3 种常见原因。
8. 口述：Go 服务内存持续增长的排查思路（heap profile + goroutine profile + 三类原因）。
9. 口述：`df` 满但 `du` 对不上怎么处理（deleted 文件 + lsof + 截断 fd）。
10. 口述：网络不通的分层排查（本机 → DNS → 路由 → 端口 → 应用 → 抓包），并说清 refused 和超时的区别。
11. 口述：Docker 镜像分层原理 + 多阶段构建 + 层缓存顺序 + 为什么容器内要监听 0.0.0.0。
12. 口述：`iostat` 的 `%util`/`await` 含义；`uptime` 负载怎么和核数比较。
13. 项目：把「刷机平台 Docker 部署 + 磁盘写满」「数据中台 goroutine 泄漏定位」两个故事各讲 1 分钟。
14. 反问准备：问面试官「线上问题的响应机制是怎样的？有没有 on-call 轮值？」「部署是运维统一管还是开发自己上？」
