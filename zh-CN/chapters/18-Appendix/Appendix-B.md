# 附录 B. 启用大页（Huge Pages）

## Windows

要在 Windows 上使用大页（huge pages），需要启用 `SeLockMemoryPrivilege` [安全策略](https://docs.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/lock-pages-in-memory)。可以通过 Windows API 以编程方式实现，也可以通过安全策略图形界面（GUI）操作。

1. 点击开始 &rarr; 搜索"secpol.msc"并启动。
2. 在左侧选择"本地策略"（Local Policies）&rarr;"用户权限分配"（User Rights Assignment），然后双击"锁定内存中的页面"（Lock pages in memory）。

![Windows security: Lock pages in memory](../../img/appendix-C/WinLockPages.png)

3. 添加您的用户并重启计算机。

4. 使用 [RAMMap](https://docs.microsoft.com/en-us/sysinternals/downloads/rammap) 工具在运行时检查大页是否被使用。

在代码中使用大页的方法如下：

```cpp
void* p = VirtualAlloc(NULL, size, MEM_RESERVE | 
                                   MEM_COMMIT | 
                                   MEM_LARGE_PAGES,
                       PAGE_READWRITE);
...
VirtualFree(ptr, 0, MEM_RELEASE);
```

## Linux

在 Linux 操作系统上，有两种方式在应用程序中使用大页：显式大页（Explicit Huge Pages）和透明大页（Transparent Huge Pages）。

### Explicit Huge Pages {.unnumbered .unlisted}

显式大页可以在系统启动时或应用程序启动前预先分配。若要永久修改，使 Linux 内核在启动时分配 128 个大页，可执行以下命令：

```bash
$ echo "vm.nr_hugepages = 128" >> /etc/sysctl.conf
```

若要在系统启动后显式分配 128 个大页，可使用以下命令：

```bash
$ echo 128 > /proc/sys/vm/nr_hugepages
```

可以在 `/proc/meminfo` 中观察到效果。注意这是系统级视图，而非按进程显示：

```bash
$ watch -n1 "cat /proc/meminfo  | grep huge -i"
AnonHugePages:      2048 kB
ShmemHugePages:        0 kB
FileHugePages:         0 kB
HugePages_Total:     128    <== 128 huge pages allocated
HugePages_Free:      128
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:          262144 kB <== 256MB of space occupied
```

可以通过在调用 `mmap` 时设置 `MAP_HUGETLB` 标志来在代码中使用显式大页（[完整示例](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/map_hugetlb.c)[^25]）：

```cpp
void ptr = mmap(nullptr, size, PROT_READ | PROT_WRITE,
                MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0);
...
munmap(ptr, size);
```

其他替代方式包括：

* 使用挂载的 `hugetlbfs` 文件系统的文件进行 `mmap`（[示例代码](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/hugepage-mmap.c)[^26]）。
* 使用 `SHM_HUGETLB` 标志的 `shmget`（[示例代码](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/hugepage-shm.c)[^27]）。

### Transparent Huge Pages {.unnumbered .unlisted}

要允许应用程序在 Linux 上使用透明大页（Transparent Huge Pages，THP），需确保 `/sys/kernel/mm/transparent_hugepage/enabled` 的值为 `always` 或 `madvise`。前者启用系统级 THP，后者允许用户代码控制哪些内存区域应使用 THP，从而避免消耗过多内存资源。以下是使用 `madvise` 方式的示例：

```cpp
void ptr = mmap(nullptr, size, PROT_READ | PROT_WRITE | PROT_EXEC,
                MAP_PRIVATE | MAP_ANONYMOUS, -1 , 0);
madvise(ptr, size, MADV_HUGEPAGE);
...
munmap(ptr, size);
```

可以在 `/proc/meminfo` 的 `AnonHugePages` 字段观察系统级效果：

```bash
$ watch -n1 "cat /proc/meminfo  | grep huge -i" 
AnonHugePages:     61440 kB     <== 30 transparent huge pages are in use
HugePages_Total:     128
HugePages_Free:      128        <== explicit huge pages are not used
```

此外，可以通过查看特定进程的 `smaps` 文件来观察应用程序如何使用显式大页（EHP）和/或透明大页（THP）：

```bash
$ watch -n1 "cat /proc/<PID_OF_PROCESS>/smaps"
```

[^25]: MAP_HUGETLB example - [https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/map_hugetlb.c](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/map_hugetlb.c).
[^26]: Mounted `hugetlbfs` filesystem - [https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/hugepage-mmap.c](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/hugepage-mmap.c).
[^27]: SHM_HUGETLB example - [https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/hugepage-shm.c](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/hugepage-shm.c).
