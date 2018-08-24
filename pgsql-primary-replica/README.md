Postgres 高可用集群部署方案
=======
方案整合基于[crunchy-containers](https://crunchydata.github.io/crunchy-containers/),结合阿里云使用
## 部署

``` bash
helm install pgsql-primary-replica --name pgsql-primary-replica --set primary.pv.volumeId=xxxx
```
primary.pv.volumeId 为阿里云盘ID

## 说明
目前的组成结构为：

* pgpool-II deployment
* primary deployment
* replica statefulset

create the following in your Kubernetes cluster:

 * Create a service named *primary*
 * Create a service named *replica*
 * Create a service named *pgpool*
 * Create a deployment named *primary*
 * Create a statefulset named *replica*
 * Create a deployment named *pgpool*
 * Create a persistent volume named *primary-pv* and a persistent volume claim of *primary-pvc*
 * Create a secret pgprimary-secret
 * Create a secret pgpool-secrets
 * Create a configMap pgpool-pgconf

## 自定义配制文件
根目录下：

* pool-configs
    - pgpool.conf
    - pool_hba.conf
    - pool_passwd
* primary-configs
    - pg_hba.conf
    - postgresql.conf
    - setup.sql

如果使用repo的方式安装，由于没有找到helm install替换这些文件的方式，目前有两个解决方案：

1. install完成后手动修改相应的configMap和secret的内容
2. helm fetch pgsql-primary-replica --untar 下源文件，修改相应的配件文件，采用 helm install ./ 的方式安装

## postgres 用户名和密码修改
如果是通过pgpool连接数据库的话，必须在pool_passwd配制postgres的用户名和密码
### 部署前
可以设置默认用户postgres和PG_USER对应用户的密码

1. 修改values.yaml中PG_ROOT_PASSWORD或者PG_PASSWORD的值，
2. 使用pgpool的pg_md5 指令生成密码（pg_md5 --md5auth --username= ）
3. 修改pool-configs/pool_passwd中的用户名和密码

### 部署后修改
先获取postgres的用户名和md5密码
``` bash
$ kubectl get pod -n thinker-production
NAME                       READY     STATUS    RESTARTS   AGE
pgpool-67987c5c67-gjr5h    1/1       Running   0          3d
primary-6f9f488df8-6xjjd   1/1       Running   0          3d
replica-0                  1/1       Running   0          3d

# 进入primary pod
$ kubectl exec -it primary-6f9f488df8-6xjjd  -n thinker-production  sh

$ psql

#执行sql查找md5
$ select usename,passwd from pg_shadow;  
   usename   |               passwd                
-------------+-------------------------------------
 postgres    | md5dc5aac68d3de3d518e49660904174d0c
 primaryuser | md576dae4ce31558c16fe845e46d66a403c
 testuser    | md599e8713364988502fa6189781bcf648f
(3 rows)
```
pool_passwd 的格式为
``` bash
postgres:md5dc5aac68d3de3d518e49660904174d0c
```
k8s secret 的values为bash46,转换
``` bash
$ echo -n 'postgres:md5dc5aac68d3de3d518e49660904174d0c' | base64
cG9zdGdyZXM6bWQ1ZGM1YWFjNjhkM2RlM2Q1MThlNDk2NjA5MDQxNzRkMGM=
```
编辑 pgpool-secrets 替换 pool_passwd 值
``` bash
kubectl edit secret pgpool-secrets  -n thinker-production
```
编辑 pgpool deploy,触发滚动更新
``` bash
kubectl edit deploy pgpool  -n thinker-production
# 一般可以修改一下 terminationGracePeriodSeconds 的值
```
## 参数说明
| Parameter  | Description  |Default|
| -----------------------    | ---------------------------------- | ---------------------------------------------------------- |
| `.container.port`| 端口| `5432`|
| `.container.storage`| 存储大小| `20G`|
| `.primary.pv.volumeId`| master 阿里云盘id||
| `.primary.pvc.mode`| PVC Access Modes，如是 ReadWriteMany 需要文件类型的支持（阿里NAS盘可以），可以做到primary 高可用|ReadWriteOnce|
| `.config.PG_USER`| 用户库的用户名|testuser|
| `.config.PG_DATABASE`| 用户库名|userdb|
| `.config.PG_PASSWORD`| 用户库的密码|testuser|
| `.config.PG_PRIMARY_USER`| 主从复制的用户名 |primaryuser|
| `.config.PG_PRIMARY_PASSWORD`| 主从复制的密码|testuser|
| `.config.PG_ROOT_PASSWORD`| root用户postgres的密码|testuser|