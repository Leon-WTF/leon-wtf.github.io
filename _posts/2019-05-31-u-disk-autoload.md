---
title: "U盘自动加载"
category: Linux
tag: linux
---
### 设置devlabel规则 ###
每次USB插入时，linux创建的设备文件可能不一样，有时候是/dev/sdb1有时候是/dev/sdc1,可以用fdisk -l命令查看。所以需要为usb建立label不管设备文件是哪个，都有一个固定的label的连接指向它。
先用***udevadm info -a -p  $(udevadm info -q path -n /dev/sdb1)***查看U盘信息，再在***/etc/udev/rules.d/***下面新建一个文件如：***11-usb-serial.rules***，写入：

```
DRIVERS=="usb", SYMLINK+="usb"
```
如果希望做更多的限制，可以参考[Writing udev rules](http://www.reactivated.net/writing_udev_rules.html)
这样每次U盘插入时就会生成一个如下的软连接：
```
lrwxrwxrwx 1 root root 4 May 31 12:34 /dev/usb -> sdb1
```

### 设置自动加载 ###
安装autofs工具：***sudo yum install autofs***
配置***/etc/auto.master***：
```
#
# Sample auto.master file
# This is a 'master' automounter map and it has the following format:
# mount-point [map-type[,format]:]map [options]
# For details of the format look at auto.master(5).
#
/home/aqui/mnt /etc/auto.usb
#
# NOTE: mounts done from a hosts map will be mounted with the
#       "nosuid" and "nodev" options unless the "suid" and "dev"
#       options are explicitly given.
#
#/net   -hosts
#
# Include /etc/auto.master.d/*.autofs
# The included files must conform to the format of this file.
#
+dir:/etc/auto.master.d
#
# Include central master map if it can be found using
# nsswitch sources.
#
# Note that if there are entries for /net or /misc (as
# above) in the included master map any keys that are the
# same will not be seen as the first read key seen takes
# precedence.
#
+auto.master
```
新建***/etc/auto.usb***：
```
usb -fstype=auto,rw,nosuid,nodev :/dev/usb
```
这样每次U盘插入时就会自动mount到/home/aqui/mnt/usb
还可以修改***/etc/autofs.conf***里的***timeout***来设置mount point空闲多少秒之后自动umount
最后，输入以下命令设置开机自启和重启autofs：
```
sudo systemctl enable autofs
sudo systemctl restart autofs
```