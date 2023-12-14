---
title: reset-centos
date: 2023-09-06 23:59:17
tags:
---

# 将Windows重装为centos/7
##### 制作启动盘
1. 现在镜像https://mirrors.aliyun.com/centos-vault/centos/7.8.2003/isos/x86_64/
2. 直接下载.iso就行
3. 下载U盘启动盘制作工具：UltraISO，一路选择试用
4. 运行 UltraISO，选择试用，选择主界面菜单栏里的[文件] → [打开]，选择你刚下载好的 CentOS 7 镜像
5. 选择菜单栏里的 [启动] → [写入硬盘映像]
6. 此时如果出现：“设备忙，关闭正在运行的应用程序”等字样时，是因为U盘没有被完全格式化
7. 那么，管理员运行Windows Terminal或者cmd，然后`diskpart`，然后`list disk`，然后`select disk 1`，然后`clean`，这就将U盘格式化成功了
8. 然后重复4和5步骤，就可以写入成功了

##### 重装系统
1. F12进入BIOS系统，我的电脑一般不自动识别U盘，需要启动一轮硬件检查，然后检查过程中可以直接中断，然后再次重启，F12后就会发现我的U盘(Sumsang字样)
2. 选择U盘启动，选择`Test this mode & Install CentOS 7`
2. 此时，出现了`Warning: /dev/root does not existed`，是因为找不到U盘挂载的位置，需要额外配置
3. 在当前最下边命令行输入：`dracut:/# ls /dev`，查询所有文件夹。此时，如果只有一个硬盘挂载的情况下，默认U盘挂载的位置是`sdb4`，如果多个硬盘的电脑，就要借助搜索引擎了
4. 然后`dracut:/# reboot`，出现选择`Test this mode & Install CentOS 7`的页面
5. 键盘输入`e`，进入编辑，将`vmlinuz initrd=initrd.img inst.stage2=hd:LABEL=CentOS\* rd.live.check quiet`改为`vmlinuz initrd=initrd.img inst.stage2=hd:/dev/sd4 quiet`，然后按键F10保存，继续安装
6. 成功！

##### 配置网络
1. 默认的CentOS是网络服务不启动的，我们要配置其开机启动，插上网线
2. 查看网卡信息`ip addr`，看看哪个网卡没有配置，一般是第二个`p8p1`
3. 然后`cd /etc/sysconfig/network-scripts/ifcfg-p8p1`，将`ONBOOT=no`改为`ONBOOT=yes`
4. 重启服务`systemctl restart network`
5. 然后`ping www.baidu.com`如果不行，就重启一下电脑
