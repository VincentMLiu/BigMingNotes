# 部署Filebeat

# 安装部署

安装包如下：

filebeat-7.16.2-linux-x86_64.tar.gz



所有节点重复下列操作：

## 0. 创建elasticsearch用户和组

```shell
groupadd elasticsearch    
useradd elasticsearch -g elasticsearch
```



## 1. 解压安装包

```shell
tar xzvf filebeat-7.17.2-linux-x86_64.tar.gz
```



## 2. 配置filebeat.yml

```yaml
#输出的ES配置
output.elasticsearch:
  hosts: ["myEShost:9200"]
  username: "filebeat_internal"
  password: "YOUR_PASSWORD"
```

