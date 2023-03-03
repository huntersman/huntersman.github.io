---
title: "Linux命令之iostat"
categories:
  - Linux
tags:
  - Linux
---

`iostat`命令通常用于监视系统IO

命令格式为`iostat 参数 时间 次数`

参数包括：
- -C显示CPU使用情况
- -d显示磁盘使用情况
- -k以KB为单位显示
- -m以M为单位显示
- -N显示磁盘阵列信息
- -n显示NFS使用情况
- -p显示磁盘和分区的情况
- -t显示终端和CPU的信息
- -x显示详细信息
- -V显示版本信息

使用示例一：
以KB为单位显示磁盘详细使用情况，每2秒刷新一次
`iostat -d -x -k 2`
```bash
Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sdb               0.00    30.50    0.00   22.50     0.00   216.00    19.20     0.00    0.07    0.00    0.07   0.29   0.65
sdg               0.00     0.00    0.00  324.50     0.00  1298.00     8.00     0.35    1.33    0.00    1.33   1.67  54.25
sdd               0.00     0.00    0.00  280.50     0.00  1122.00     8.00     0.26    1.20    0.00    1.20   1.54  43.10
```
属性说明：
- rrqm/s：每秒进行merge的读操作数目。即rmerge/s
- wrqm/s：每秒进行merge的写操作数目。即wmerge/s
- r/s：每秒完成的读IO设备次数。即rio/s
- w/s：每秒完成的写IO设备次数。即wio/s
- rkB/s：每秒读K字节数。是rsect/s的一半，因为每个扇区大小为512字节
- wkB/s：每秒写K字节数。是wsect/s的一半
- avgrq-sz：平均每次设备I/O操作的数据大小 (扇区)。
- avgqu-sz：平均I/O队列长度
- await：平均每次设备I/O操作的等待时间 (毫秒)
- r_await：读请求平均耗时
- w_await：写请求平均耗时
- svctm：平均每次设备I/O操作的服务时间 (毫秒)
- %util：一秒中有百分之多少的时间用于 I/O 操作，即被io消耗的cpu百分比

备注：如果 %util 接近 100%，说明产生的I/O请求太多，I/O系统已经满负荷，该磁盘可能存在瓶颈。如果 svctm 比较接近 await，说明 I/O 几乎没有等待时间；如果 await 远大于 svctm，说明I/O 队列太长，io响应太慢，则需要进行必要优化。如果avgqu-sz比较大，也表示有当量io在等待。

参考资料
- [https://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/iostat.html](https://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/iostat.html)
- [https://superuser.com/questions/1399903/what-is-the-relationship-of-await-r-await-and-w-await-in-iostat-x-command](https://superuser.com/questions/1399903/what-is-the-relationship-of-await-r-await-and-w-await-in-iostat-x-command)