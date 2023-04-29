# 1.下载 VirtualBox并安装

```
https://www.virtualbox.org/wiki/Downloads
```

# 2.下载Rocky Linux

选择 Rocky-9.1-x86_64-dvd.iso 镜像
官网
```
https://rockylinux.org/download
```
阿里云镜像服务
```
https://mirrors.aliyun.com/rockylinux/9.1/isos/x86_64/?spm=a2c6h.25603864.0.0.70c556799nMQLA
```
# 3.创建虚拟机镜像

- 选择Rocky Linux镜像文件
![](https://files.mdnice.com/user/34714/97ec7be6-0cc9-4469-90db-c3b155e0a4ff.png)


![](https://files.mdnice.com/user/34714/60d4e873-9a20-422d-b87f-f0949e99a906.png)

- 设置用户/密码、Hostname、Domain Name
![](https://files.mdnice.com/user/34714/02dbc084-c286-4ce0-b5ea-6f6ae40b613d.png)

- 设置内存、处理器个数
![](https://files.mdnice.com/user/34714/ae023587-323b-47a7-af9d-ae315b8a3cc7.png)

- 设置磁盘空间
![](https://files.mdnice.com/user/34714/d6b3e445-a5b2-4413-83b9-4e87cb95c8b0.png)


- 挂载光驱文件

![](https://files.mdnice.com/user/34714/1028c9dd-d520-4452-9ef6-79a1f50c3791.png)

- 启动虚拟机

![](https://files.mdnice.com/user/34714/d5371c30-d1d2-40e8-acd1-5712beedc655.png)

# 4.安装RockyLinux

- 选择语言

![](https://files.mdnice.com/user/34714/6600d25e-9451-497d-a847-2981f374aa69.png)

- 选择安装的硬盘和设置root用户的密码

![](https://files.mdnice.com/user/34714/1f5f1ce2-1cc7-4a04-8205-0d4247aad712.png)

选择硬盘

![](https://files.mdnice.com/user/34714/d1e417ff-54e2-4a83-92ca-d00d23a2af15.png)


设置密码


![](https://files.mdnice.com/user/34714/cb41bfda-ff8a-4d19-ac94-0686ab7631c5.png)

# 5.VirtualBox安装增强功能

去VirtualBox官网下载文件：VBoxGuestAdditions_7.0.6.iso。文件名中的 **7.0.6** 是VirtuaBox的版本号，可以选择和原来的VirtuaBox的版本号保持一致的。
```
https://download.virtualbox.org/virtualbox
```
文件VBoxGuestAdditions_7.0.6.iso下载到本地以后，进入Virtual Box，进行如下图的操作：

设置->存储->控制器：IDE->蓝色齿轮->选择虚拟盘->选择刚刚下载好的VBoxGuestAdditions_7.0.6.iso

![](https://files.mdnice.com/user/34714/f1987d14-33e7-40fc-b226-53a345c4504b.png)

启动Linux找到挂载的VBoxGuestAdditions_7.0.6.iso

![](https://files.mdnice.com/user/34714/d617b3f2-4687-4622-99f1-ed06ff1fc362.png)

在终端进入该目录，执行 VBoxLinuxAdditions.run，切换到root用户，执行命令如下：

```
cd /run/media/huangjinjin/VBox_GAs_7.0.6
sudo ./VBoxLinuxAdditions.run
```

# 6.网络配置
- VirtualBox设置网络为桥接
在VirtualBox选中安装的RockyLinux系统，点击设置，再选择网络，勾选“启用网络连接”，并在连接方式中选择“桥接网卡”

![](https://files.mdnice.com/user/34714/d2afbf8f-50fb-4502-ba02-3d2d078877db.png)

- linux中设置ip，子网掩码，网关

打开文件(如果ifcfg-enp0s3不存在直接创建)
```
vi /etc/sysconfig/network-scripts/ifcfg-enp0s3 
```

```
DEVICE=enp0s3 #网卡名称，必须和ifcfg-eth0后面的eth0一样
HWADDR=08:00:27:77:AE:95 #网卡的MAC地址，默认的
TYPE=Ethernet #类型
UUID=c031fded-f139-4751-9357-d873107480ed #uuid，不重要
ONBOOT=yes #是否默认启动此接口的意思，填yes
NM_CONTROLLED=yes #是否接受其他软件的网络管理
BOOTPROTO=statics #ip获取的方式，填static时需要手动设置
IPADDR=192.168.10.108 #设置的ip地址
NETMASK=255.255.255.0 #设置的子网掩码
GATEWAY=192.168.10.1 #设置的默认网管
```

重点关注网关（GATEWAT），可以看到和我们的主机网关一致（若不一致则修改为一致）：


![](https://files.mdnice.com/user/34714/2c1cf550-5a9a-4f20-8e61-9f29857bf8fa.png)

需要注意的地方，此处的IPADDR，NERTMASK, GATEWAY需要跟你的Windows系统设置的ip相对向，所以需要查看win的网络设置，进行设置。

- 重启网络

查看网络状态
```
systemctl status NetworkManager
```
开机启动网络
```
systemctl enable NetworkManager
```
取消开机启动网络
```
systemctl disable NetworkManager
```
开启网络
```
systemctl start NetworkManager
```
 重启网络
```
systemctl restartNetworkManager
```
关闭网络
```
[root@rockylinux tmp]#    systemctl    stop    NetworkManager
```


# 7. 遇到的问题

- 报cdrom被占用，这个时候需要将当期的虚拟光盘中的盘片清除，也就是取消勾选。
- 再次点击安装增强，如果提示无法打开virtualbox下面的一个xxx.iso的话，去网站上搜索对应virtualbox版本的缺失的这个xxx.iso，并放到提示的目录下。
- 再次点击安装增强，如不提示错误，证明安装成功了
- 以linux系统为例，需要把光盘中的内容mount到可以操作的文件夹下，比如在/tmp/下
**以下命令都在root用户下操作**
```
cd /tmp
mkdir cdrom
```
创建一个cdrom的文件夹，然后使用命令：
```
mount /dev/cdrom cdrom
```
然后`cd cdrom`到 cdrom 文件下，执行
```
./VBoxLinuxAdditions.run
```

- 如果出现
```
kernel headers not found for target kernel
```
需要执行
```
yum update kernel -y
yum install kernel-headers kernel-devel gcc make -y
```
然后执行重启
```
reboot
```
- 再次执行1~4步骤，如果还有问题比如
```
“VirtualBox Guest Additions: Kernel headers not found for target kernel
4.19.0-6-amd64. Please install them and execute
/sbin/rcvboxadd setup”
```
改完之后日志里面没有错，输出的结果里只剩下一个挂载失败：
```
ValueError: File context for /opt/VBoxGuestAdditions-6.30.1/other/mount.vboxsf already defined
```
在root用户下执行：
```
semanage fcontext -d /opt/VBoxGuestAdditions-/other/mount.vboxsf
restorecon /opt/VBoxGuestAdditions-/other/mount.vboxsf
```
然后重启
```
reboot
```
再重复1~4的操作即可。


```
参考：
https://blog.csdn.net/OrdinaryMatthew/article/details/124040107
https://blog.csdn.net/arthaslonely/article/details/122654186
https://www.bbsmax.com/A/8Bz8GYekdx/
https://dandelioncloud.cn/article/details/1561230165407920130
```







