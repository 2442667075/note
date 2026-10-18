# Linux 命令完整目录

共 20 个模块，按基础学习 → 系统管理 → 网络排障 → 嵌入式专项 → 开发调试组织。

## 第一部分：Linux 基础与 Shell

### 01. 登录、Shell 会话与终端

- `login`、`logout`、`exit`
    
- `bash`、`sh`
    
- `history`
    
- `tty`、`stty`
    
- `clear`、`reset`
    
- `script`
    
- `screen`、`tmux`
    

### 02. 输出、变量、环境与 Shell 脚本

- `echo`、`printf`
    
- `set`、`unset`
    
- `export`、`env`
    
- `source`、`.`
    
- `alias`、`unalias`
    
- `read`
    
- `eval`（了解即可，注意安全风险）
    
- `shift`
    
- `getopts`
    
- `trap`
    
- `test`、`[`、`[[`
    
- `true`、`false`
    
- `sleep`
    
- `seq`
    
- `xargs`
    

### 03. 命令帮助与查找

- `man`
    
- `info`
    
- `whatis`
    
- `apropos`
    
- `help`
    
- `type`
    
- `command`
    
- `which`
    
- `whereis`
    

## 第二部分：文件与目录

### 04. 文件、目录与路径操作

- `pwd`、`cd`
    
- `ls`
    
- `mkdir`、`rmdir`
    
- `touch`
    
- `cp`、`mv`
    
- `rm`
    
- `ln`
    
- `readlink`
    
- `realpath`
    
- `tree`
    
- `basename`、`dirname`
    
- `install`
    

### 05. 文件权限、属主与属性

- `chmod`
    
- `chown`
    
- `chgrp`
    
- `umask`
    
- `stat`
    
- `file`
    
- `getfacl`、`setfacl`
    
- `chattr`、`lsattr`
    
- `getfattr`、`setfattr`
    

### 06. 文件查找、校验与批量处理

- `find`
    
- `locate`、`updatedb`
    
- `which`、`whereis`
    
- `stat`
    
- `md5sum`
    
- `sha256sum`
    
- `cksum`
    
- `rename`
    
- `xargs`
    
- `mktemp`
    

## 第三部分：文件系统、磁盘与归档

### 07. 磁盘、分区、挂载与文件系统

- `df`
    
- `du`
    
- `sync`
    
- `mount`、`umount`
    
- `findmnt`
    
- `lsblk`
    
- `blkid`
    
- `fdisk`
    
- `parted`
    
- `mkfs`
    
- `fsck`
    
- `mkswap`、`swapon`、`swapoff`
    
- `losetup`
    
- `fuser`
    

### 08. 打包、压缩与镜像操作

- `tar`
    
- `gzip`、`gunzip`
    
- `bzip2`、`bunzip2`
    
- `xz`、`unxz`
    
- `zip`、`unzip`
    
- `zstd`
    
- `dd`
    
- `cpio`
    
- `file`
    
- `hexdump`、`od`
    

## 第四部分：文本处理与数据分析

### 09. 文件内容查看与日志阅读

- `cat`
    
- `tac`
    
- `more`
    
- `less`
    
- `head`
    
- `tail`
    
- `nl`
    
- `watch`
    

### 10. 文本处理三剑客与常用工具

- `grep`、`egrep`、`fgrep`
    
- `sed`
    
- `awk`
    
- `cut`
    
- `paste`
    
- `sort`
    
- `uniq`
    
- `tr`
    
- `join`
    
- `comm`
    
- `diff`
    
- `patch`
    
- `wc`
    
- `split`
    
- `csplit`
    
- `fmt`
    
- `fold`
    
- `iconv`
    
- `dos2unix`、`unix2dos`
    

## 第五部分：进程、任务与性能监控

### 11. 进程管理与调度

- `ps`
    
- `top`
    
- `pstree`
    
- `pgrep`
    
- `pkill`
    
- `kill`
    
- `killall`
    
- `nice`
    
- `renice`
    
- `nohup`
    
- `jobs`、`fg`、`bg`
    
- `wait`
    
- `disown`
    
- `timeout`
    
- `chrt`
    
- `taskset`
    

### 12. 系统资源与性能监控

- `uptime`
    
- `free`
    
- `uname`
    
- `hostname`
    
- `dmesg`
    
- `vmstat`
    
- `iostat`
    
- `mpstat`
    
- `sar`
    
- `pidstat`
    
- `pmap`
    
- `smem`（如已安装）
    
- `lsof`
    
- `strace`
    
- `perf`
    
- `time`
    
- `ulimit`
    
- `sysctl`
    

### 13. 进程间通信与资源限制

- `ipcs`
    
- `ipcrm`
    
- `ulimit`
    
- `prlimit`
    
- `lslocks`
    
- `flock`
    
- `fuser`
    

## 第六部分：用户、权限与系统服务

### 14. 用户与用户组管理

- `whoami`
    
- `who`
    
- `w`
    
- `users`
    
- `last`
    
- `lastlog`（视系统支持）
    
- `id`
    
- `groups`
    
- `useradd`、`adduser`
    
- `usermod`
    
- `userdel`
    
- `passwd`
    
- `groupadd`
    
- `groupmod`
    
- `groupdel`
    
- `su`
    
- `sudo`
    
- `chage`
    

### 15. 启动、服务、日志与计划任务

- `init`
    
- `systemctl`
    
- `service`
    
- `journalctl`
    
- `logger`
    
- `logread`
    
- `dmesg`
    
- `reboot`
    
- `poweroff`
    
- `shutdown`
    
- `crontab`
    
- `at`
    
- `date`
    
- `hwclock`
    
- `watchdog`
    
- `start-stop-daemon`
    

## 第七部分：网络与远程传输

### 16. 网络配置与连通性排查

- `ip`
    
- `ifconfig`
    
- `route`
    
- `ip neigh`
    
- `arp`（旧工具）
    
- `arping`
    
- `ping`
    
- `traceroute`
    
- `tracepath`
    
- `mtr`
    
- `ethtool`
    
- `brctl`
    
- `bridge`
    
- `udhcpc`
    
- `dhclient`
    
- `hostname`
    

### 17. 端口、套接字、DNS 与抓包

- `ss`
    
- `netstat`
    
- `nc`、`ncat`
    
- `telnet`
    
- `host`
    
- `nslookup`
    
- `dig`
    
- `getent`
    
- `tcpdump`
    
- `tshark`（如已安装）
    
- `lsof`
    
- `fuser`
    

### 18. 远程登录、文件传输与网络请求

- `ssh`
    
- `sshd`
    
- `scp`
    
- `sftp`
    
- `ftp`
    
- `lftp`
    
- `wget`
    
- `curl`
    
- `rsync`
    
- `rclone`（了解即可，视需求安装）
    
- `tftp`
    
- `socat`（进阶）
    

## 第八部分：嵌入式 Linux 专项

### 19. 内核、驱动、设备与硬件调试

内核模块

- `lsmod`
    
- `insmod`
    
- `rmmod`
    
- `modprobe`
    
- `modinfo`
    
- `depmod`
    

设备与内核信息

- `dmesg`
    
- `ls -l /dev`
    
- `cat /proc/devices`
    
- `cat /proc/interrupts`
    
- `cat /proc/iomem`
    
- `cat /proc/ioports`
    
- `cat /proc/cpuinfo`
    
- `cat /proc/meminfo`
    
- `cat /proc/mounts`
    
- `findmnt`
    

硬件总线与外设

- `lsusb`
    
- `lspci`
    
- `i2cdetect`
    
- `i2cget`
    
- `i2cset`
    
- `i2cdump`
    
- `gpiodetect`
    
- `gpioinfo`
    
- `gpioget`
    
- `gpioset`
    
- `devmem`
    
- `hexdump`
    
- `od`
    

存储与 Flash

- `mtdinfo`
    
- `flash_erase`
    
- `flashcp`
    
- `nandwrite`
    
- `nanddump`
    
- `ubinfo`
    
- `ubiattach`
    
- `ubidetach`
    
- `ubimkvol`
    
- `ubirmvol`
    

> 注意：MTD/UBI 命令需要相应工具和内核支持；不同 Flash 类型的擦除、写入方式不同。执行擦写前必须确认设备节点、分区布局和备份方案。

### 20. 交叉编译、程序调试与构建工具

编译与构建

- `gcc`
    
- `g++`
    
- `make`
    
- `cmake`
    
- `ninja`
    
- `pkg-config`
    
- `ar`
    
- `ranlib`
    
- `ld`
    
- `as`
    
- `strip`
    

ELF 与二进制分析

- `file`
    
- `readelf`
    
- `objdump`
    
- `nm`
    
- `size`
    
- `strings`
    
- `addr2line`
    
- `objcopy`
    
- `xxd`
    

程序调试

- `gdb`
    
- `gdbserver`
    
- `strace`
    
- `ltrace`
    
- `valgrind`（目标平台和架构支持时）
    
- `perf`
    
- `coredumpctl`（systemd 环境）
    
- `eu-stack`（如已安装）
    

交叉编译工具链示例

- `arm-linux-gnueabihf-gcc`
    
- `aarch64-linux-gnu-gcc`
    
- `arm-linux-gnueabihf-readelf`
    
- `aarch64-linux-gnu-objdump`
    
- `aarch64-linux-gnu-gdb`
    

