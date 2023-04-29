# 卸载 docker


1. 杀死docker有关的容器

```
docker kill $(docker ps -a -q)
```

2. 删除所有docker容器

```
docker rm $(docker ps -a -q)
```

3. 删除所有docker镜像

```
docker rmi $(docker images -q)
```

4. 停止 docker 服务

```
systemctl stop docker
```

5. 删除docker相关存储目录

```
rm -rf /etc/docker
rm -rf /run/docker
rm -rf /var/lib/dockershim
rm -rf /var/lib/docker
```

**如果删除不掉，则先umount**

```
umount /var/lib/docker/devicemapper
```

6. 查看系统已经安装了哪些docker包

```
yum list installed | grep docker
```

出现以下安装包的信息

```
containerd.io.x86_64 1.4.12-3.1.el7 @docker-ce-stable
docker-ce.x86_64 3:20.10.12-3.el7 @docker-ce-stable
docker-ce-cli.x86_64 1:20.10.12-3.el7 @docker-ce-stable
docker-ce-rootless-extras.x86_64 20.10.12-3.el7 @docker-ce-stable
docker-scan-plugin.x86_64 0.12.0-3.el7 @docker-ce-stable
```

7. 卸载已安装的docker相关包

```
yum remove containerd.io.x86_64 docker-ce.x86_64  \
           docker-ce-cli.x86_64                   \
           docker-ce-rootless-extras.x86_64       \
           docker-scan-plugin.x86_64
```

提示选择，直接输入“y”；然后回车即可。

再次查看

```
yum list installed | grep docker
```

不再出现相关安装包，说明卸载成功。

# 安装 docker

官网文档

```
https://docs.docker.com/desktop/install/linux-install/
```

1. 安装依赖

```
yum install -y yum-utils       \
  device-mapper-persistent-data \
  lvm2
```

2. 设置rpm包的yum源

```
yum-config-manager        \
    --add-repo            \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

3. 安装docker

```
yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

如果需要安装特定版本。先查看可用docker版本

```
$yum list docker-ce --showduplicates | sort -r

docker-ce.x86_64  3:18.09.1-3.el7                     docker-ce-stable
docker-ce.x86_64  3:18.09.0-3.el7                     docker-ce-stable
docker-ce.x86_64  18.06.1.ce-3.el7                    docker-ce-stable
docker-ce.x86_64  18.06.0.ce-3.el7                    docker-ce-stable
```

通过其完整的软件包名称安装特定版本，该软件包名称是软件包名称（docker-ce）加上版本字符串（第二列），从第一个冒号（:）一直到第一个连字符，并用连字符（-）分隔。例如：docker-ce-18.09.1。

```
yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io
```

# 启动 docker

```
#启动
systemctl start docker
#重启
systemctl restart docker
#停止
systemctl stop docker
```

# 卸载 docker-compose

1. 查看docker-compose命令所在目录

```
whereis docker-compose
```

查询以下信息

```
/usr/bin/docker-compose
```

2. 删除docker-compose

```
rm -rf /usr/bin/docker-compose
```

# 安装 docker-compose

docker-compose源码仓库

```
https://github.com/docker/compose/releases
```

1. 安装

官网文档

```
https://docs.docker.com/compose/install/other/
```

下载docker-compose

```
curl -SL https://github.com/docker/compose/releases/download/v2.12.2/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
```

2. 授权

```
chmod +x /usr/local/bin/docker-compose
```

3. 创建软连接

```
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

4. 测试

```
docker-compose --version
```

