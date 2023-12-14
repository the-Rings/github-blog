---
title: k8s-cluster
date: 2023-09-13 12:34:29
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
6. 修改时间时区`timedatectl set-timezone Asia/Shanghai`
```shell
# 时间同步修正
sudo yum install ntp
sudo ntpdate ntp3.aliyun.com
# 查看
date 
```

# Vagrant Install
virtual box和vagrant配合比vmware更加有效率
官网即可下载两个工具，不需要配置任何环境变量（注意打开BIOS中的虚拟化功能才能使用virtual box，一般都是默认打开的）
1. vagrant可以快速下载镜像并创建虚拟机，先去vagrantup上搜索镜像，找到镜像名，找一个干净的文件夹，执行命令
```shell
cd target_dir
vagrant init centos/7 
# vagrant init centos/7 https://mirrors.ustc.edu.cn/centos-cloud/centos/7/vagrant/x86_64/images/CentOS-7.box
# 加上国内的mirros加速下载
```

2. 启动并进入虚拟机
```shell
vagrant up
vagrant ssh
# 如果修改了配置文件等相关信息，就要使用`vagrant reload`优雅地重启虚拟机

```
此时，可以在virtual box工具中看到这个虚拟机，但是整个过程中无需在virtual box进行操作

3. 替换国内repo
```shell
sudo mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup

sudo curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo

sudo yum makecache

# 修改时区
timedatectl set-timezone Asia/Shanghai
# 时间同步修正
sudo yum install ntp
sudo ntpdate ntp3.aliyun.com
# 查看
date 
```

4. (弃用)编辑Vagrantfile，使用网络地址转换(NAT)模式，是虚拟机成为宿主机的一部分
执行vagrant up后就会在当前文件夹生成一个Vagrantfile文件，这个文件不能挪动位置
为虚拟机指定ip能使主机和虚拟机便捷通信，首先查看主机ipconfig的配置"VirtualBox Host-Only Network"项对应的ip，
VirtualBox的ip与虚拟机的ip必须保持在同一个网段，编辑Vagrantfile文件的行，假设VirtualBox的ip为192.168.56.1
```
config.vm.network "private_network", ip: "192.168.56.10"
```

5. 如果要进行局域网访问，使虚拟机成为一台在局域网中独立的机器(桥接模式)，放弃第4步
```
config.vm.network "public_network", bridge: "Intel(R) Ethernet Connection (16) I219-V"
```
注意：这里的bridge网桥，通过在宿主机上执行`ipconfig /all`，查看当前的网络适配器，如果你使用网线接入，安装了VirtaulBox，那么你会看到两个网络适配器，一个是宿主机真实的网卡，一个是虚拟网卡
```
以太网适配器 以太网 2:

   连接特定的 DNS 后缀 . . . . . . . :
   描述. . . . . . . . . . . . . . . : Intel(R) Ethernet Connection (16) I219-V
   物理地址. . . . . . . . . . . . . : 98-8F-E0-64-DE-43
   DHCP 已启用 . . . . . . . . . . . : 否
   自动配置已启用. . . . . . . . . . : 是
   IPv6 地址 . . . . . . . . . . . . : 2408:8256:2e80:838:db54:b1d5:b1ef:458(首选)
   临时 IPv6 地址. . . . . . . . . . : 2408:8256:2e80:838:98c3:3136:3976:e052(首选)
   本地链接 IPv6 地址. . . . . . . . : fe80::c40b:5832:8a7e:4841%14(首选)
   IPv4 地址 . . . . . . . . . . . . : 192.168.1.2(首选)
   子网掩码  . . . . . . . . . . . . : 255.255.255.0
   默认网关. . . . . . . . . . . . . : fe80::1%14
                                       192.168.1.1
   DHCPv6 IAID . . . . . . . . . . . : 127438816
   DHCPv6 客户端 DUID  . . . . . . . : 00-01-00-01-2B-B4-BB-F6-98-8F-E0-64-DE-43
   DNS 服务器  . . . . . . . . . . . : fe80::1%14
                                       22.8.33.100
                                       114.114.114.114
   TCPIP 上的 NetBIOS  . . . . . . . : 已启用

以太网适配器 以太网 3:

   连接特定的 DNS 后缀 . . . . . . . :
   描述. . . . . . . . . . . . . . . : VirtualBox Host-Only Ethernet Adapter
   物理地址. . . . . . . . . . . . . : 0A-00-27-00-00-0C
   DHCP 已启用 . . . . . . . . . . . : 否
   自动配置已启用. . . . . . . . . . : 是
   本地链接 IPv6 地址. . . . . . . . : fe80::6824:2664:4b:f79e%12(首选)
   IPv4 地址 . . . . . . . . . . . . : 192.168.1.4(首选)
   子网掩码  . . . . . . . . . . . . : 255.255.255.0
   默认网关. . . . . . . . . . . . . :
   DHCPv6 IAID . . . . . . . . . . . : 805961767
   DHCPv6 客户端 DUID  . . . . . . . : 00-01-00-01-2B-B4-BB-F6-98-8F-E0-64-DE-43
   DNS 服务器  . . . . . . . . . . . : 22.8.33.100
                                       114.114.114.114
   TCPIP 上的 NetBIOS  . . . . . . . : 已启用

无线局域网适配器 WLAN:

   媒体状态  . . . . . . . . . . . . : 媒体已断开连接
   连接特定的 DNS 后缀 . . . . . . . :
   描述. . . . . . . . . . . . . . . : Realtek RTL8852BE WiFi 6 802.11ax PCIe Adapter
   物理地址. . . . . . . . . . . . . : 9C-2F-9D-AA-1D-FF
   DHCP 已启用 . . . . . . . . . . . : 是
   自动配置已启用. . . . . . . . . . : 是

....

```
使用`netsh interface show interface`查看，
```
管理员状态     状态           类型             接口名称
-------------------------------------------------------------------------
已启用            已连接            专用               以太网 2
已启用            已连接            专用               以太网 3
已启用            已断开连接          专用               WLAN
```
此时，如果拔掉网线，发现“以太网2”断开连接，但是“以太网3”仍然是保持连接。
"Intel(R) Ethernet Connection (16) I219-V" 是您计算机上的真实物理网卡，而 "VirtualBox Host-Only Ethernet Adapter" 则是虚拟机软件 VirtualBox 创建的虚拟网络适配器。
"Intel(R) Ethernet Connection (16) I219-V" 是一种常见的物理网卡，用于连接计算机到物理网络，例如通过有线连接到路由器或交换机。
"VirtualBox Host-Only Ethernet Adapter" 是 VirtualBox 虚拟机软件创建的虚拟网络适配器，用于虚拟机之间或者虚拟机与主机之间的通信。所以，它不会断。

6. Vagrant File完整实例
```
Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"
  config.vm.box_url = "https://mirrors.ustc.edu.cn/centos-cloud/centos/7/vagrant/x86_64/images/CentOS-7.box"

  config.vm.network "public_network", bridge: "Intel(R) Ethernet Connection (16) I219-V"
end
```

7. 测试局域网连通性
- 虚拟机执行`ip addr`查看ip，一般是`eth1`中的`inet`
- 无论是，宿主机ping虚拟机，还是虚拟机ping局域网的其他机器，应该都是通的。

# Docker install
下载路径https://docs.docker.com > Get Docker > Docker Engine > Install On CentOS
读文档只选择重要的步骤：
1. 删除旧docker
```shell
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```
2. 配置repo
```shell
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```
3. 安装docker
```shell
sudo yum -y install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```
4. 启动docker
```shell
sudo systemctl start docker
# 配置开机启动
sudo systemctl enable docker
```
5. 配置阿里云docker镜像加速
登录阿里云 > 控制台 > 产品与服务 > 容器服务 > 容器镜像服务 > 镜像工具 > 镜像加速器 > CentOS > 配置镜像加速
```shell
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://alsecg6o.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

# 部署Master节点
sudo yum remove kubeadm kubectl kubelet conntrack-tools cri-tools kubernetes-cni libnetfilter_cthelper libnetfilter_cttimeout libnetfilter_queue socat 

sudo passwd root

cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

0. 执行命令时加上--v=5可以输出更多日志

1. sudo yum install -y kubelet-1.22.10 kubeadm-1.22.10 kubectl-1.22.10 --disableexcludes=kubernetes

2. sudo yum -y install docker-ce-20.10.8 docker-ce-cli-20.10.8 containerd.io-1.4.10 docker-compose-plugin

3. systemctl enable kubelet.service & systemctl start kubelet

4. swapoff -a

5. 将docker的cgroups driver改为systemd，使用`docker info`，查看是否cgroup是否为systemd
```shell
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://alsecg6o.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```
5. kubeadm reset & swapoff -a
6. kubeadm init --apiserver-advertise-address=192.168.1.16 --image-repository=registry.aliyuncs.com/google_containers

7. 每次启动，都要执行以下三个命令
rm -rf .kube
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
8. kubectl -n kube-system apply -f ./kube-flannel.yml
8. cd /etc/kubernetes/manifests，查看kube-apiserver.yaml中的ip是否都是192.168.1.16
9. kubectl get node -n kube-system，发现NotReady
10. journalctl -f -u kubelet.service，查看是否有报错
```
Sep 10 09:57:58 localhost.localdomain kubelet[12866]: E0910 09:57:58.965423   12866 kubelet.go:2376] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"
Sep 10 09:58:01 localhost.localdomain kubelet[12866]: I0910 09:58:01.621500   12866 cni.go:239] "Unable to update cni config" err="no networks found in /etc/cni/net.d"
```

11. kubectl get pods --all-namespaces，检查发现其中有两个pending
12. kubectl describe pod -n kube-system coredns-7f6cbbb7b8-wgt7f，深入查看pod的详细信息
13. kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml  安装一个网络插件（很重要）
```国外大神的总结
There are several points to remember when setting up the cluster with "kubeadm init" and it is clearly documented on the Kubernetes site kubeadm cluster create:

"kubeadm reset" if you have already created a previous cluster
Remove the ".kube" folder from the home or root directory
(Also stopping the kubelet with systemctl will allow for a smooth setup)
Disable swap permanently on the machine, especially if you are rebooting your linux system
And not to forget, install a pod network add-on according to the instructions provided on the add on site (not Kubernetes site) --> kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml

Follow the post initialization steps given on the command window by kubeadm.
If all these steps are followed correctly then your cluster will run properly.

And don't forget to do the following command to enable scheduling on the created cluster:

kubectl taint nodes --all node-role.kubernetes.io/master-
About how to install from behind proxy you may find this useful:
```
14. kubectl taint nodes --all node-role.kubernetes.io/master-
15. 如果token过期或者忘记：kubeadm token create --print-join-command，重新生成，让worker加入



# 部署Worker节点
sudo yum remove kubeadm kubectl kubelet conntrack-tools cri-tools kubernetes-cni libnetfilter_cthelper libnetfilter_cttimeout libnetfilter_queue socat 

sudo passwd root

cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

1. sudo yum install -y kubelet-1.22.10 kubeadm-1.22.10 kubectl-1.22.10 --disableexcludes=kubernetes

2. sudo yum -y install docker-ce-20.10.8 docker-ce-cli-20.10.8 containerd.io-1.4.10 docker-compose-plugin

3. systemctl enable kubelet.service & systemctl start kubelet

4. kubeadm join 192.168.1.16:6443 --token wsflz8.af9y1q44c5obd3ja --discovery-token-ca-cert-hash sha256:cc172ac5ced3fefd7514a50e5153b011f649bb5b8ec5c005c36d9f3431b1250f
5. 