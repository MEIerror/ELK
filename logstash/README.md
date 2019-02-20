### 14-ia.conf grok匹配
```
%{TIMESTAMP_ISO8601:iadate}.*- %{DATA:MN}，设备编号：%{WORD:num}.*message =(?<mess>[0-9a-f]{,222})

解析日志为：
2019-01-10 00:00:04 INFO  Decoder:38 - 监听到数据，设备编号：DT8200201811YH9888 设备地址 (0x000002FF: nio socket, server, /221.178.125.140:5106 => /192.168.1.125:20000) message =5aa5053067000300000044543832303
```
