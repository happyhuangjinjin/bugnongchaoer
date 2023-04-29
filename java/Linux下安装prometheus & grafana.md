# 1 安装prometheus
## 1.1 下载prometheus
下载地址
```
https://prometheus.io/download/#prometheus
```
下载
```
wget https://github.com/prometheus/prometheus/releases/download/v2.42.0/prometheus-2.42.0.linux-amd64.tar.gz
```

## 1.2 安装

```
# 新建目录，并进入目标目录
mkdir -p /middleware/prometheus && cd /middleware/prometheus

# 解压
tar -vxzf /root/prometheus-2.42.0.linux-amd64.tar.gz -C /middleware/prometheus

cd /middleware/prometheus
mv prometheus-2.42.0.linux-amd64 prometheus
```

## 1.3 启动prometheus
前台启动prometheus
```
./prometheus --config.file=prometheus.yml
```
后台启动prometheus，并且重定向输入日志到当前目录的prometheus.out
```
nohup ./prometheus --config.file=prometheus.yml >> /middleware/prometheus/prometheus/prometheus.out 2>&1 &
```

## 1.4 访问prometheus
prometheus启动完后，监听端口为9090
```
http://192.168.10.223:9090/
```

# 2 安装grafana

## 2.1 下载grafana

下载地址
```
https://grafana.com/
https://grafana.com/grafana/download?pg=get&plcmt=selfmanaged-box1-cta1&edition=oss
```

下载
```
wget https://dl.grafana.com/oss/release/grafana-9.4.2.linux-amd64.tar.gz

```

## 2.2 安装

```
# 新建目录，并进入目标目录
mkdir -p /middleware/grafana && cd /middleware/grafana

# 解压
tar -vxzf /middleware/grafana-9.4.2.linux-amd64.tar.gz -C /middleware/grafana

cd /middleware/grafana
mv grafana-9.4.2 grafana

```

## 1.3 启动grafana

grafana默认的配置文件在$GRAFANA_HOME/conf/defaults.ini 中，该文件中的内容不要修改。复制一份，命名为`grafana.ini`；自定义的配置文件通过 --config 来指定加载路径

```
cp /middleware/grafana/grafana/confdefaults.ini     \ 
    /middleware/grafana/grafana/conf/grafana.ini
```

后台启动grafana

```
nohup /middleware/grafana/grafana/bin/grafana-server \
-config "/middleware/grafana/grafana/conf/grafana.ini" \
-homepath "/middleware/grafana/grafana" \
-pidfile "/middleware/grafana/grafana/grafana.pid" web \
>> /middleware/grafana/grafana/grafana.out 2>&1 &
```
- config: 自定义的配置文件的路径
- homepath: homepath的路径，否则程序启动不了
- pidfile: pid文件的路径

## 1.4 访问grafana
grafana启动完后，监听端口为3000；默认的用户名和密码是： `admin/admin`
```
http://192.168.10.223:3000/
```

```
https://blog.csdn.net/qq_43386944/article/details/122727712
https://www.cnblogs.com/ezops/p/16607845.html
```