# 对proc中的进程文件了解

## /proc/pid/maps
maps文件记录了进程的虚拟内存空间信息。
https://stackoverflow.com/questions/1401359/understanding-linux-proc-id-maps

也可以用命令pmap查看

## /proc/pid/smaps
smaps文件详细记录了关于虚拟内存空间到物理内存空间的映射信息。
http://blog.chinaunix.net/uid-23062171-id-5754717.html