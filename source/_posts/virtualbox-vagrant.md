---
title: virtual box使用
date: 2022-01-22 16:22:12
tags:
- Virtual Box
---

# Install
virtual box和vagrant配合比vmware更加有效率
官网即可下载两个工具，不需要配置任何环境变量（注意打开BIOS中的虚拟化功能才能使用virtual box，一般都是默认打开的）

# Useage
1. vagrant可以快速下载镜像并创建虚拟机，先去vagrantup上搜索镜像，找到镜像名，找一个干净的文件夹，执行命令
```shell
cd target_dir
vagrant init centos/7 
# vagrant init centos/7 https://mirrors.ustc.edu.cn/centos-cloud/centos/7/vagrant/x86_64/images/CentOS-7.box
# 加上国内的mirros加速下载
```
2. 启动并进入虚拟机
```
vagrant up
vagrant ssh
```
可以在virtual box工具中看到这个虚拟机，但是整个过程中无需在virtual box进行操作
3. 替换国内repo
```
sudo mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup

sudo curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo

sudo yum makecache
```
4. 编辑Vagrantfile
执行vagrant up后就会在当前文件夹生成一个Vagrantfile文件，这个文件不能挪动位置
为虚拟机指定ip能使主机和虚拟机便捷通信，首先查看主机ipconfig的配置"VirtualBox Host-Only Network"项对应的ip，
VirtualBox的ip与虚拟机的ip必须保持在同一个网段，编辑Vagrantfile文件的行，假设VirtualBox的ip为192.168.56.1
```
config.vm.network "private_network", ip: "192.168.56.10"
```
