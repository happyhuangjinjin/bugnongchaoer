docker镜像官网：https://hub.docker.com

# 1. 拉取镜像

```
docker pull 镜像名 
docker pull 镜像名:tag
```

不加 tag (版本号) 即拉取 docker 仓库中，该镜像的最新版本 latest，加 :tag 则是拉取指定版本。

例如

```shell
docker pull alpine:3.13
```

# 2. 查看服务器中docker 镜像列表

```
docker images
```

# 3. 运行镜像

```
docker run 镜像名
docker run 镜像名:Tag
```

例如

```
 docker run -it alpine:3.13 sh
```

