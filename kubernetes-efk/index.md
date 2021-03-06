# kubernetes EFK



# kubernetes EFK

## 初始化环境

```
# 增加max_map_count
echo 'vm.max_map_count=262144' >> /etc/sysctl.conf

sysctl -p

```

## 配置 namespace

```
vi  logging-namespace.yaml

---
apiVersion: v1
kind: Namespace
metadata:
  name: logging

```



## 配置 elasticsearch

```
# 增加 elasticsearch-deployment.yaml

curl -O https://raw.githubusercontent.com/jicki/kuberneres/master/efk/elasticsearch-deployment.yaml

# 这里需要按需修改

# JVM 配置
"-Xms4g -Xmx4g"

# volume 配置
      volumes:
      - name: es-volume
        persistentVolumeClaim:
          claimName: efk-claim

```

```
# 增加 elasticsearch-service.yaml 文件

curl -O https://raw.githubusercontent.com/jicki/kuberneres/master/efk/elasticsearch-service.yaml

```

## 配置 fluentd

```
# 增加 fluentd-daemonset.yaml
# td-agent 这一段 配置输入，请按需配置

curl -O https://raw.githubusercontent.com/jicki/kuberneres/master/efk/fluentd-daemonset.yaml

```

## 配置 kibana

```
# 增加  kibana-deployment.yaml 

curl -O https://raw.githubusercontent.com/jicki/kuberneres/master/efk/kibana-deployment.yaml



# 增加  kibana-service.yaml
curl -O https://raw.githubusercontent.com/jicki/kuberneres/master/efk/kibana-service.yaml

```


## 导入 yaml

```
# 导入 

kubectl apply -f logging-namespace.yaml

kubectl apply -f fluentd-daemonset.yaml

kubectl apply -f elasticsearch-deployment.yaml

kubectl apply -f elasticsearch-service.yaml

kubectl apply -f kibana-deployment.yaml

kubectl apply -f kibana-service.yaml


# 查看导入

kubectl get all --namespace=logging -o wide

```

## 测试

```
# elasticsearch 管理端口为 30200
http://node-ip:30200/


# kibana 访问页面 30601 

http://node-ip:30601

# 1. 选择 @timestamps 点击 create 

# 2. curl -O https://raw.githubusercontent.com/jicki/kuberneres/master/efk/dashboards/elk-v1.json

# 3. 选择 management > Saved Object > Import > elk-v1.json


# 4. 选择 Dashboard 勾选 ELK 并点击 ELK，等待加载。


```
![EFK Dashboard][1]


  [1]: https://jicki.cn/img/posts/efk/1.png


