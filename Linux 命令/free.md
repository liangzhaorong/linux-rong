## 常见用法
注：如下为 Centos 7 新版的 free。旧版的 free 输出三行。

直接输入 free，输出内容以 KB 为单位：
```
# free 
              total        used        free      shared  buff/cache   available
Mem:        1015476      474696      320476       19156      220304      363340
Swap:        999420           0      999420
```
加上 -m 选项，则以 MB 为单位：
```
# free -m
              total        used        free      shared  buff/cache   available
Mem:            991         486         290          18         214         331
Swap:           975           0         975
```
加上 -h 选项，则以人类易读的方式显示：
```
# free -h
              total        used        free      shared  buff/cache   available
Mem:           991M        489M        287M         18M        215M        329M
Swap:          975M          0B        975M
```

## 慎用 -g 选项
加上 -g 选项表示输出内容以 GB 为单位显示，这样虽然可以增加可读性，但存在隐患：-g 选项会采用向下取整的方式显示内存容量，如原本 3031MB 的内存容量，换算后变为 2.96GB，最后会显示成 2GB。

## 精确查看内容容量
使用 -k、-m、-g 等选项，free 命令的输出策略都是向下取整，所以向查看一台服务器最精确的内容容量，则使用 -b 选项，这会让 free 命令以最小的 byte 为单位显示内容使用情况：
```
# free -b
              total        used        free      shared  buff/cache   available
Mem:     1039847424   519319552   287358976    19615744   233168896   338829312
Swap:    1023406080           0  1023406080
```

## buffers 和 cached
buffers 和 cached 都是属于内存的一部分，它们和空闲内存、已用内存一起构成了整个内存容量。

硬盘和内存的读写速度有着本质的区别，DDR4 内存读写速度大概在 60GB/s 量级，而 SSD 固态硬盘的读写速度大概为 600MB/s。

当有大量数据要从内存写入硬盘时，为了防止读写数据速度的巨大差距导致的时间等待，在内存中创造了一个叫做 buffers 的内存区域，数据不再直接 "缓慢" 地写入硬盘，而是先写入 buffers 中，然后再在后台慢慢地写入硬盘中，这样对于应用程序来说，就不需要再等待数据完全写入硬盘，就可以去做其他事情了。

系统设计的一个重要原则就是 "要尽量减少内存从磁盘读数据的次数"。从硬盘读取的内容，往往会暂存在 cache 里，下次如果又读到了，那么就不用再找硬盘要了，而是直接从 cache 里拿。

cache 和 cached 的关系：cache 即表示那块专用的内存区域中的内容，而 cached 则表示被缓存住的。

free 命令中的 buffers 和 cached 值，是读取自 /proc/meminfo 文件中的对应值。而 /proc 中的绝大部分内容是 Linux 内核来控制和更新的。

从源码分析可以得出更专业的结论：
- buffers 是块设备 I/O 相关的缓存页。
- cached 是普通文件相关的缓存页。`
