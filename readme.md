# NVMe SSD 完整性能测试流程 README

## 测试说明

本文档为 **NVMe 企业级SSD 全套稳态性能测试标准流程**，包含磁盘初始化格式化、顺序读写预处理、随机读写预处理、正式性能测试全链路步骤。

测试设备：`/dev/nvme2n1`

测试工具：`nvme-cli`、`fio`

测试规范：

- 所有测试均开启 `direct=1` 绕过系统缓存，测试硬件真实裸盘性能

- 所有任务开启随机地址不重复、全覆盖写入，保证稳态测试准确性

- 区分**长时间预处理（打稳态、清空SLC缓存）**和**正式性能采集**

## 前置依赖

执行测试前需安装依赖工具：

```Plain Text
# CentOS / RHEL
yum install fio nvme-cli -y

# Ubuntu / Debian
apt install fio nvme-cli -y
```

## 完整测试步骤（按顺序执行）

> **注意：所有命令必须 root 权限执行，测试会清空磁盘所有数据，请确认磁盘无业务数据！**
> 
> 

### 1、磁盘初始化格式化（恢复出厂闪存映射）

完整重建磁盘逻辑块、闪存映射表，无安全擦除，快速初始化磁盘状态，为基准测试做准备。

```Plain Text
nvme format /dev/nvme2n1 -s 0 -l 0 -i 0 -p 0 -m 1 -f
```

参数释义：

- `-s 0`：不开启安全擦除

- `-l 0`：逻辑块 512B（通用标准）

- `-i 0`：关闭PI数据校验

- `-m 1`：完整格式化，重建全盘映射表

- `-f`：强制执行，无需交互确认

### 2、顺序写预处理（1h 稳态打盘）

128K 大块顺序写，长时间压测打满磁盘缓存，使SSD进入稳态性能区间。

```Plain Text
fio --ioengine=libaio --direct=1 --time_based=1 --end_fsync=1 --ramp_time=5 --norandommap=1 --randrepeat=0 --group_reporting=1 --overwrite=1 --name=nvme2n1_preheat_seq_1 --runtime=3600 --filename=/dev/nvme2n1 --numjobs=1 --iodepth=64 --rw=write --bs=128K
```

### 3、顺序读预处理（30min 预热）

128K 顺序读预热，消除磁盘冷启动性能波动，统一测试基线。

```Plain Text
fio --ioengine=libaio --direct=1 --time_based=1 --end_fsync=1 --ramp_time=5 --norandommap=1 --randrepeat=0 --group_reporting=1 --overwrite=1 --name=nvme2n1_preheat_read_1 --runtime=1800 --filename=/dev/nvme2n1 --numjobs=1 --iodepth=256 --rw=read --bs=128K
```

### 4、顺序写正式性能测试（5min）

```Plain Text
fio --ioengine=libaio --direct=1 --time_based=1 --end_fsync=1 --ramp_time=5 --norandommap=1 --randrepeat=0 --group_reporting=1 --overwrite=1 --name=nvme2n1_seq_write_0 --runtime=300 --filename=/dev/nvme2n1 --numjobs=1 --iodepth=64 --rw=write --bs=128K
```

### 5、顺序读正式性能测试（5min）

```Plain Text
fio --ioengine=libaio --direct=1 --time_based=1 --end_fsync=1 --ramp_time=5 --norandommap=1 --randrepeat=0 --group_reporting=1 --overwrite=1 --name=nvme2n1_seq_read_0 --runtime=300 --filename=/dev/nvme2n1 --numjobs=1 --iodepth=64 --rw=read --bs=128K
```

### 6、4K随机写预处理（6h 深度稳态打盘）

4K小块随机写长时间压测，耗尽SLC缓存、模拟业务极限稳态场景，是SSD性能衰减核心测试项。

```Plain Text
fio --ioengine=libaio --direct=1 --time_based=1 --end_fsync=1 --ramp_time=5 --norandommap=1 --randrepeat=0 --group_reporting=1 --overwrite=1 --name=nvme2n1_preheat_randwrite_1 --runtime=21600 --filename=/dev/nvme2n1 --numjobs=4 --iodepth=32 --rw=randwrite --bs=4K
```

### 7、4K随机写正式性能测试（5min）

```Plain Text
fio --ioengine=libaio --direct=1 --time_based=1 --end_fsync=1 --ramp_time=5 --norandommap=1 --randrepeat=0 --group_reporting=1 --overwrite=1 --name=nvme2n1_rand_write_0 --runtime=300 --filename=/dev/nvme2n1 --numjobs=4 --iodepth=32 --rw=randwrite --bs=4K
```

### 8、4K随机读预处理（1h 预热）

高并发、大队列深度随机读预热，抹平磁盘冷热数据差异。

```Plain Text
fio --ioengine=libaio --direct=1 --time_based=1 --end_fsync=1 --ramp_time=5 --norandommap=1 --randrepeat=0 --group_reporting=1 --overwrite=1 --name=nvme2n1_preheat_randread_1 --runtime=3600 --filename=/dev/nvme2n1 --numjobs=4 --iodepth=256 --rw=randread --bs=4K
```

### 9、4K随机读正式性能测试（5min）

行业标准4K随机读高并发测试，衡量SSD核心IOPS与延迟性能。

```Plain Text
fio --ioengine=libaio --direct=1 --time_based=1 --end_fsync=1 --ramp_time=50 --norandommap=1 --randrepeat=0 --group_reporting=1 --overwrite=1 --name=nvme2n1_rand_read_0 --runtime=300 --filename=/dev/nvme2n1 --numjobs=16 --iodepth=256 --rw=randread --bs=4K
```

## 通用FIO参数统一说明

|参数|作用|
|---|---|
|\-\-ioengine=libaio|Linux原生异步IO，贴合生产业务场景|
|\-\-direct=1|绕过PageCache，测试裸盘真实硬件性能|
|\-\-time\_based=1|按时间计时运行，而非按数据量|
|\-\-norandommap=1|不预分配随机地址，节省内存，适配全盘压测|
|\-\-randrepeat=0|随机地址不重复，实现全盘全覆盖读写|
|\-\-group\_reporting=1|聚合多进程测试结果，输出统一报表|
|\-\-overwrite=1|强制覆盖旧数据，保证测试有效性|

## 测试顺序规范

初始化格式化 → 顺序稳态预热 → 顺序性能测试 → 随机稳态预热 → 随机性能测试

严格遵循 **先预热、后测试** 原则，避免冷盘、缓存干扰导致测试数据失真。

## 注意事项

- 本测试为**破坏性测试**，执行前务必确认磁盘无业务数据、无重要文件

- 长时间预热任务（1h/6h）请勿中途中断，否则无法达到稳态测试效果

- 测试期间禁止挂载、读写该磁盘，保证测试环境纯净

- 所有测试完成后，可通过fio日志分析带宽、IOPS、平均延迟、99\.99%长尾延迟指标
