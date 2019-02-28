## 需求
- 线上某数据接入服务频繁掉线，导致数据上报异常，前端接口调用不到数据。
- 研发人员需要定位数据数据异常时，哪类数据包没有上报。

数据上传情况，数据包类型可以通过日志查找。所以使用ELK对日志内容进行收集整理。

## 数据采集
日志内容格式如下
```
2019-02-28 11:01:15 INFO  Decoder:38 - 监听到数据，设备编号：DT8100201805YH2036 设备地址 (0x00000003: nio socket, server, /117.132.192.107:41787 => /192.168.1.125:20000
) message =5aa503308a000d0000004454383130303230313830355948323033360500000007000900be030000070008000000000007000a000000000007000b000000000008000900c803000000aa3d3d0022263dc0014d3d80fd643dc06f523dc0a75e3dc0ab833d2061863d000000000000000000000000000000000200000000000000130000000000010013000000f84e775ca3990000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
2019-02-28 11:01:15 INFO  Decoder:45 - 监听到数据，设备编号：DT8100201805YH2036 消息处理正常
2019-02-28 11:01:15 INFO  Encoder:47 - 下发数据，设备编号：DT8100201805YH2036 设备地址 (0x00000003: nio socket, server, /117.132.192.107:41787 => /192.168.1.125:20000)2
message =5aa5035024000d0000004454383130303230313830355948323033364f4b0000000000000000fb4e775cd9cb
```
其中包含了很多无用数据，需要去除。

通过filebeat对日志内容进行收集并做简单过滤。

```
- type: log
  enabled: true
  paths:
    - /root/work/ia-lng-dgb/ia-log/lng2.log
  fields:
    type: lng-ia
  tail_files: true
  include_lines: ['设备地址']
  
  
output.logstash:
  hosts: ["192.168.1.82:5044"]
```

- 从`/root/work/ia-lng-dgb/ia-log/lng2.log`路径下收集日志；
- 添加`[fields][type]`字段，方便判断；
- 日志中包含“设备地址”信息的为需要收集的日志；
- 将手机的数据发送至对应主机的logstash中。

## 过滤存储
logstash中对数据格式进行过滤，
```
filter {
  if [fields][type] == "lng-ia" {
    grok {
      match => { "message" => "%{TIMESTAMP_ISO8601:iadate}.*- %{DATA:MN}，设备编号：%{WORD:num}.*message =(?<mess>[0-9a-f]{,222})" }
      remove_field => ["message"]
        }

#        date{
#                match=>["iadate","yyyy-MM-dd HH:mm:ss"]
#       }
}
}

output {
if [fields][type] == "lng-ia" {
  elasticsearch {
    hosts => ["localhost"]
    manage_template => false
    index => "lng-ia-%{+YYYY.MM}"
    document_type => "%{[@metadata][type]}"
  }
}
}
```
- 因为message字段信息比较长，所只截取前222位。(后期发现统计时研发只需要前8位,可适当改短)
过滤后的结果如下：
```
{
  "MN": "监听到数据",
  "iadate": "2019-02-28 11:01:15",
  "num": "DT8100201805YH2036",
  "mess": "5aa503308a000d00000044"
}
```
## kibana展示
在kibana页面加入`lng-ia`索引.
界面如下
![image](https://i.loli.net/2019/02/28/5c77901e4085a.png)

#### 图表展示
根据`mess`字段前几位判断不同类型的数据包

![image](https://i.loli.net/2019/02/28/5c77919e309ad.png)

上下发数据：
![image](https://i.loli.net/2019/02/28/5c77925d35f49.png)

最终展示图：
![all](https://i.loli.net/2019/02/28/5c7792d6bc273.png)

还可根据需要进一步绘图。