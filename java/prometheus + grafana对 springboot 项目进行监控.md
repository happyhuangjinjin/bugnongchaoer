# 1.prometheus接入springboot

prometheus安装后，在安装目录有一个默认的配置文件`prometheus.yml`

```
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]
```

默认配置了一个job_name，监控prometheus本身。需要增加一个监控springboot项目
```
  - job_name: "custom_spring_boot"
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ["localhost:9595"]
```

![](https://files.mdnice.com/user/34714/1e4d4398-cefd-4a1c-bf8a-cc62b400588b.png)

- **metrics_path** 默认采集metrics的路径是`/metrics`；需要改成`/actuator/prometheus`
- **scheme** 默认是http；如果是https需要自定义配置
- **targets** 获取metrics的地址和端口列表

# 2.访问prometheus

```
http://127.0.0.1:9090/
```
![](https://files.mdnice.com/user/34714/80f79e3c-f9b2-47b7-bb41-4aaf7e9aecda.png)

出现自定义需要监控的springboot端点列表

![](https://files.mdnice.com/user/34714/3fe6886e-c145-47f3-bb5e-7bf1b221910f.png)

在首页，可以查询各种不同的指标
![](https://files.mdnice.com/user/34714/2a5d408d-7a47-41ff-bc4f-d02eef97dc7b.png)

比如查询`custom_http_request_time_seconds_count`指标

![](https://files.mdnice.com/user/34714/b98ebbf2-4030-4338-9b37-961cf2434ce5.png)

# 3.grafana接入prometheus

访问
```
http://127.0.0.1:3000/
```

配置数据源


![](https://files.mdnice.com/user/34714/f079376f-b153-4b02-bb9b-2eb9b69bf145.png)

添加一个数据


![](https://files.mdnice.com/user/34714/dcc89c40-f561-40f8-a7c4-4047114f1dcb.png)

选择prometheus

![](https://files.mdnice.com/user/34714/5e82dc6b-8723-4cfa-acc2-67e06c8acca0.png)

设置名称和prometheus服务地址

![](https://files.mdnice.com/user/34714/7d6825bf-fa23-41b0-971d-c399b46ccca6.png)


![](https://files.mdnice.com/user/34714/5e1ba190-90c2-4a1b-a83b-f14397f23129.png)

# 4.配置仪表盘


![](https://files.mdnice.com/user/34714/1312eb16-04c5-435b-ba4a-85a1334c93ca.png)

点击`Add a new panel`；新建一个`Panel`


![](https://files.mdnice.com/user/34714/3acdd234-9ba9-4705-b3c4-e0d4410b3671.png)

平均时间查询

```
sum by(api) (rate(custom_http_request_time_seconds_count{job="custom_spring_boot", api="/order"}[5m]))
```
![](https://files.mdnice.com/user/34714/cba45548-a1e0-4671-878a-028aa432ad32.png)

保存，最终显示


![](https://files.mdnice.com/user/34714/f53b2eec-91c5-4ea2-bd05-231a7d34a63d.png)
