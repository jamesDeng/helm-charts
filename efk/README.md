k8s集群日志方案 EFK
=======
Filebeat + Elasticsearch + Kibana,部署方案思路请参考[K8s日志处理](https://jamesdeng.github.io/2018/08/27/k8s%E6%97%A5%E5%BF%97%E5%A4%84%E7%90%86.html)
## 部署
建议源码部署，Filebeat的配制文件如需自定义必须源码部署
``` bash
#简单安装
helm install --name efk ./
#指定values文件
helm install --name -f values.yaml efk ./
```
如需要指定namespace,不要使用 helm 命令的 --namesapce 参数，正确方式：
``` bash
helm install --name efk --set namesapce=kube-system ./
```
ps:由于要做权限，使用values.yaml的配制做namespace

## 说明
目前的组成结构为：
* Kibana deployment
* Filebeat DaemonSet
* Elasticsearch statefulset

## Elasticsearch 存储说明
如要使用阿里云盘,修改 values.yaml 文件
``` yaml
es: 
  pvc:
    #开启 pvc，创建阿里云盘
    enable: true
    storageClassName: alicloud-disk-efficiency #盘类型
    storage: 20Gi #大小
```

## kibana 访问权限说明
使用Ingress添加basic-auth认证的方式做权限

安装 htpasswd，创建用户密码
``` bash
#安装
$ yum -y install httpd-tools

#产生密码文件 auth
$ htpasswd -c auth  kibana

New password: 
Re-type new password: 
Adding password for user kibana
```
复制 auth 文件到 kibana-secrets 目录下，修改 values.yaml文件
``` yaml
kibana: 
  ingress:
    enable: true
    basicAuth: 
      enable: true # 进行 http basic-auth 授权
      authSecret: kibana-basic-auth 
      username: kibana #用户名
```

## 自定义配制文件
根目录下：
* filebeat-inputs
    - kubernetes.yml
* filebeat-config.yml
* kibana-secrets
    - auth

如果使用repo的方式安装，由于没有找到helm install替换这些文件的方式，目前有两个解决方案：

1. install完成后手动修改相应的configMap和secret的内容
2. helm fetch pgsql-primary-replica --untar 下源文件，修改相应的配件文件，采用 helm install ./ 的方式安装


## 参数说明
请参考 values.yaml文件说明