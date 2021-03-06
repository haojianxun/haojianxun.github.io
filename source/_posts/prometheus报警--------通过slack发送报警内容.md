---
title: prometheus报警--------通过slack发送报警内容
---



1.注册slack账号

打开[slack](https://slack.com/)官网

![](http://pe9685fps.bkt.clouddn.com/18-9-2/19371697.jpg)



2.邮箱注册![](http://pe9685fps.bkt.clouddn.com/18-9-2/10854855.jpg)



 3.到邮箱去填验证码

![](http://pe9685fps.bkt.clouddn.com/18-9-2/13938653.jpg)

4.填自己的名字

![](http://pe9685fps.bkt.clouddn.com/18-9-2/38520584.jpg)

5.设置密码

![](http://pe9685fps.bkt.clouddn.com/18-9-2/46543482.jpg)

6.填写相关信息

![](http://pe9685fps.bkt.clouddn.com/18-9-2/84321311.jpg)

7.填公司名称

![](http://pe9685fps.bkt.clouddn.com/18-9-2/70251603.jpg)

8.自定义url

![](http://pe9685fps.bkt.clouddn.com/18-9-2/34443446.jpg)

9下一步,可以跳过

![](http://pe9685fps.bkt.clouddn.com/18-9-2/77991713.jpg)

10.创建频道

![](http://pe9685fps.bkt.clouddn.com/18-9-2/61164320.jpg)

11安装一个应用 incomming webhooks 

![](http://pe9685fps.bkt.clouddn.com/18-9-2/82718456.jpg)



![](http://pe9685fps.bkt.clouddn.com/18-9-2/64513740.jpg)

![](http://pe9685fps.bkt.clouddn.com/18-9-2/10138998.jpg)

![](http://pe9685fps.bkt.clouddn.com/18-9-2/18903748.jpg)

![](http://pe9685fps.bkt.clouddn.com/18-9-2/81000640.jpg)







## 写报警规则

```
cd /promethues  //进入到prometheus目录下
cp prometheus.yml{,.bak}   //先备份配置文件

```

2. vim prometheus.yaml

```
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
      monitor: 'codelab-monitor'

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  - "/root/prometheus/prometheus/example.rules"     //这里填写prometheus报警规则文件路径
  # - "second.rules"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'container'         //增加一个实例

    static_configs:
      - targets: ['localhost:8080']   //目前url
   
  - job_name: 'node'   //增加一个node_exporter
    
    static_configs:
       - targets: ['192.168.200.139:9100']   //node_exporter的目标url
```

3. vim /root/prometheus/prometheus/example.rules

   ```
   ALERT InstanceDown   # alert 名字
     IF up == 0           # 判断条件
     FOR 15s             # 条件保持 15s 才会发出 alert
     LABELS { severity = "critical" }  # 设置 alert 的标签
     ANNOTATIONS {             # alert 的其他标签，但不用于标识 alert
       summary = "服务器运行状态",
       description = "服务器已经宕机超过15s.",
       username= "haojietion",
       link="xxxx"  #这个link会在报警中体现 ,点击会调到报警的内容
     }
   
   ```


## 配置alertmanager

```
cd alertmanager   //进入alertmanager目录下
cp alertmanager.yml{,.bak}   //备份配置文件
```

2. vim alertmanager.yml

   ```
   global:
       resolve_timeout: 5m
   route:
       receiver: 'default-receiver'
       group_wait: 30s
       group_interval: 1m
       repeat_interval: 1m
       group_by: ['alertname']
    
       routes:
       - match:
           severity: critical
         receiver: my-slack     #填好自定义的路由
    
   receivers:
   - name: 'my-slack'
     slack_configs:
     - send_resolved: true
       api_url: https://hooks.slack.com/services/xxxx   #这里填你安装webhook app应该的时候给的url
       channel: '#test'   #要发往那个频道
       text: "{{ range .Alerts }} {{ .Annotations.description}}\n {{end}} @{{ .CommonAnnotations.username}} <{{.CommonAnnotations.link}}| click here>"
       title: "{{.CommonAnnotations.summary}}"
       title_link: "{{.CommonAnnotations.link}}" 
    
   - name: 'default-receiver'
     slack_configs:
     - send_resolved: true
       api_url: https://hooks.slack.com/services/xxxx
       channel: '#test'
       text: "{{ .CommonAnnotations.description }} \n {{end}} {{ .CommonAnnotations.username}}"
   
   ```

3. 启动应用

   ```
   cd prometheus
   
   #启动prometheus
   
   ./prometheus -alertmanager.url=http://192.168.200.139:9100         //这里填安装alertmanager的机器,让 Prometheus 服务与 Alertmanager 进行通信 现在是安装在本机,所以填本机的ip地址,url的格式要齐全 带上http://  要不然会报错
   
   
   #之后启动alertmanager
   cd alertmanager
   ./alertmanager --config.file="alertmanager.yml   //可以指定要用的配置文件
   
   #之后启动监控自身状态的node_exporter
   cd node_exporter
   ./node_exporter
   
   
   ```


4.最后报警效果如下

![](http://pe9685fps.bkt.clouddn.com/18-9-2/43930972.jpg)

