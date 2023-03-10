---
layout: post
title: "etcd自动备份与恢复"
date:   2022-03-24
tags: [etcd]
comments: true
author: xakdwch
---

# 摘要

本文档主要介绍`etcd`集群自动备份及还原方案。

# 备份

`kubernetes`中部署的应用的信息都存放在`etcd`里面，这里面的数据非常重要，需要备份，以备不时之需。定时任务的`pod`要和`etcd`在同一个`node`上面。

## 实现原理

利用`k8s CronJob`来实现`etcd`集群的自动备份，并基于`k8s`自身特性实现了`etcd`自动备份功能的高可用性。通过以下几点来实现`etcd`备份：

- 根据备份时间策略，定时生成`etcd snapshot`。
- 通过`nodeAffinity`将`etcd`备份`CronJob`调度到`etcd`节点上运行。
- 将`etcd snapshot`统一上传到网络存储（`sftp`，`ceph对象存储`，其它）。
- 根据用户自定义的备份保留数量，只保留相应的最新备份数量。

## 示例说明

`etcd`自动备份`CronJob demo`如下所示：

```javascript
apiVersion: v1
kind: ConfigMap
metadata:
  name: cron-sftp
data:
  entrypoint.sh: |
    #!/bin/bash
    
    #variables
    sftp_user="test"
    sftp_passwd="test"
    sftp_url="sftp://192.168.x.x:1234"
    backup_dir=/etcd-backup/$CLUSTER_NAME

    # backup etcd data
    mkdir -p /snapshot
    file=etcd-snapshot-$(date +%Y%m%d-%H%M%S).db
    etcdctl --endpoints $ENDPOINTS \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    snapshot save /snapshot/$file

    # upload etcd snapshot file
    lftp -u $sftp_user,$sftp_passwd $sftp_url<<EOF
    mkdir -p $backup_dir
    cd $backup_dir
    lcd /snapshot
    put $file
    by
    EOF

    # remove the expired snapshot file
    total_num=$(lftp -u $sftp_user,$sftp_passwd $sftp_url -e "ls $backup_dir/etcd-snapshot-*.db | wc -l;by")
    if [ $total_num -gt $BACKUP_COUNTS ]; then
      expired_num=$(expr $total_num - $BACKUP_COUNTS)
      expired_files=$(lftp -u $sftp_user,$sftp_passwd $sftp_url -e "ls $backup_dir/etcd-snapshot-*.db | head -n $expired_num;by" | awk '{print $NF}')
      for f in $expired_files; do
        to_remove=${backup_dir}/${f}
        echo "start to remove $to_remove"
        lftp -u $sftp_user,$sftp_passwd $sftp_url -e "rm -f $to_remove;by"
      done
    fi

    # remove local etcd snapshot file
    rm -f /snapshot/$file
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: cron-s3
data:
  entrypoint.sh: |
    #!/bin/bash

    #variables
    ak="test"
    sk="test"
    host_base="http://10.20.x.x:7480"
    host_bucket="10.20.x.x:7480/%(bucket)"
    backup_bucket=etcd-backup

    # backup etcd data
    mkdir -p /snapshot
    file=etcd-snapshot-$(date +%Y%m%d-%H%M%S).db
    etcdctl --endpoints $ENDPOINTS \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    snapshot save /snapshot/$file

    # upload etcd snapshot file
    backup_path=s3://$backup_bucket/$CLUSTER_NAME/
    s3cmd --access_key=$ak --secret_key=$sk --host="$host_base" --host-bucket="$host_bucket" --no-ssl put /snapshot/$file $backup_path

    # remove the expired snapshot file
    total_num=$(s3cmd --access_key=$ak --secret_key=$sk --host="$host_base" --host-bucket="$host_bucket" --no-ssl ls $backup_path | wc -l)
    if [ $total_num -gt $BACKUP_COUNTS ]; then     
      expired_num=$(expr $total_num - $BACKUP_COUNTS)
      echo "-----------------------------"
      expired_files=$(s3cmd --access_key=$ak --secret_key=$sk --host="$host_base" --host-bucket="$host_bucket" --no-ssl ls $backup_path | head -n $expired_num | awk '{print $NF}')
      for f in $expired_files; do
        echo "start to remove $f"
        s3cmd --access_key=$ak --secret_key=$sk --host="$host_base" --host-bucket="$host_bucket" --no-ssl rm $f
      done
    fi

    # remove local etcd snapshot file
    rm -f /snapshot/$file
---
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: etcd-backup-sftp
spec:
 schedule: "*/2 * * * *"
 jobTemplate:
  spec:
    template:
      metadata:
       labels:
        app: etcd-backup
      spec:
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
                  nodeSelectorTerms:
                  - matchExpressions:
                    - key: node-role.kubernetes.io/master
                      operator: Exists
        containers:
        - name: etcd-backup
          image: etcd-backup:v3.4.9
          imagePullPolicy: IfNotPresent
          workingDir: /
          command: ["sh", "./entrypoint.sh"]
          env:
          - name: ENDPOINTS
            value: "192.168.x.x:2379"
          - name: ETCDCTL_API
            value: "3"
          - name: BACKUP_COUNTS
            value: "5"
          - name: CLUSTER_NAME
            value: "cluster1"
          volumeMounts:
            - mountPath: /entrypoint.sh
              name: configmap-volume
              readOnly: true
              subPath: entrypoint.sh
            - mountPath: /etc/kubernetes/pki/etcd
              name: etcd-certs
              readOnly: true
            - mountPath: /etc/localtime
              name: lt-config
            - mountPath: /etc/timezone
              name: tz-config
        volumes:
          - name: configmap-volume
            configMap:
              defaultMode: 0777
              name: cron-sftp
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
          - name: lt-config
            hostPath:
              path: /etc/localtime
          - name: tz-config
            hostPath:
              path: /etc/timezone
        hostNetwork: true
        restartPolicy: OnFailure
```

上述示例说明如下：

- 通过nodeAffinity将执行etcd备份的CrobJob调度到任意etcd节点上运行。示例中设置如下：

```js
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: node-role.kubernetes.io/etcd
    	  operator: Exists
```

- 定时任务执行策略在`CronJob`的`.spec.schedule`处设置，参考cron表达式语法。示例中`schedule: "*/2 * * * *"`代表每隔2分钟执行一次备份任务。
- 执行备份任务的`Job`镜像对应的`etcd`版本，需要和被备份的`etcd`版本一致，本方案也提供了etcd镜像的制作方法。
- 执行备份任务的`Job`通过调用`entrypoint.sh`来完成`etcd`的备份，而`entrypoint.sh`脚本通过`ConfigMap`挂载到`Job`对应的`Pod`中。
- 本例提供`SFTP`和`s3`这两种存储方案来保存`etcd`备份数据，实现方法分别对应示例中的`cron-sftp`和`cron-s3`这两个`ConfigMap`。推荐选择独立于本`k8s`集群之外的存储服务来保存备份数据，保证本k8s集群挂了后仍然可以正常获取到备份数据。如果需要支持其它存储方案，只需通过`ConfigMap`将实现存储对接的`entrypoint.sh`脚本挂载到`Job`对应的`Pod`中即可。
- 通过`volumeMounts`将`etcd`证书信息映射到执行备份任务的`Job`对应的`Pod`目录。例如，本示例中`etcd`证书位于`/etc/kubernetes/pki/etcd`目录下。
- 环境变量说明如下：

   - `ENDPOINTS`：`etcd endpoints`，根据实际情况进行配置，如果配置多个`endpoint`需通过逗号隔开。
   - `ETCDCTL_API`：`etcd API`版本，根据使用的`etcd`版本来确定，本例`ETCDCTL_API=3`。
   - `BACKUP_COUNTS`：备份数，只保留最新的备份。
   - `CLUSTER_NAME`：需要备份的本集群名称。

```js
env:
- name: ENDPOINTS
  value: "192.168.x.x:2379"
- name: ETCDCTL_API
  value: "3"
- name: BACKUP_COUNTS
  value: "5"
- name: CLUSTER_NAME
  value: "cluster1"
```



## etcd镜像制作

本示例中用于制作`etcd`镜像的`Dockerfile`如下：

```javascript
FROM python:3-alpine

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories

ARG ETCD_VERSION=v3.4.9

RUN apk add -U --no-cache curl lftp ca-certificates openssh

RUN mkdir /root/.ssh  \
    && touch /root/.ssh/config \
    && echo -e "Host *\n\tStrictHostKeyChecking no\n\tUserKnownHostsFile /dev/null\n\tKexAlgorithms +diffie-hellman-group1-sha1\n\tPubkeyAcceptedKeyTypes +ssh-rsa\n\tHostkeyAlgorithms +ssh-rsa" > /root/.ssh/config

ADD s3cmd-master.zip /s3cmd-master.zip
RUN unzip /s3cmd-master.zip -d /opt \
    && cd /opt/s3cmd-master \
    && python setup.py install \
    && rm -rf /s3cmd-master.zip

RUN curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VERSION}/etcd-${ETCD_VERSION}-linux-amd64.tar.gz -o /opt/etcd-${ETCD_VERSION}-linux-amd64.tar.gz \
    && cd /opt && tar xzf etcd-${ETCD_VERSION}-linux-amd64.tar.gz \
    && mv etcd-${ETCD_VERSION}-linux-amd64/etcdctl /usr/local/bin/etcdctl \
    && rm -rf etcd-${ETCD_VERSION}-linux-amd64*
```

说明如下：

- 基础镜像为`alpine`镜像。本示例之所以使用安装了python的`alpine`镜像，是由于存储方案支持`ceph对象存储`，`etcd`备份文件需要通过`s3cmd`来进行上传，而`s3cmd`需要`python`环境支持。
- 镜像中除了安装对应版本的`etcdctl`以外，还安装了`lftp`和`s3cmd`这两个工具，`lftp`用于上传备份文件到`SFTP`服务器，而`s3cmd`用于上传备份文件到`Ceph对象存储`。
- 另外根据实际情况，需要对`ssh`配置进行设置，保证可以正常访问`sftp`服务端。

# 恢复

获取到`etcd`备份文件后，然后依次在每个`etcd`节点上执行`etcd`数据恢复操作。

## 准备备份文件

首先获取到备份文件，以本示例来进行说明。

从`SFTP`服务端下载备份文件，确保本机已经安装了`SFTP`客户端：

```javascript
# sftp -P 1022 test@192.168.x.x
test@192.168.x.x's password: 
Connected to 192.168.x.x.
sftp> cd /etcd-backup/cluster1/
sftp> ls -l
-rw-rw-rw-   1 user     group    42291232 Mar 24 07:42 etcd-snapshot-20220324-154204.db
-rw-rw-rw-   1 user     group    42291232 Mar 24 07:44 etcd-snapshot-20220324-154405.db
-rw-rw-rw-   1 user     group    42291232 Mar 24 07:46 etcd-snapshot-20220324-154605.db
-rw-rw-rw-   1 user     group    42291232 Mar 24 07:48 etcd-snapshot-20220324-154806.db
-rw-rw-rw-   1 user     group    42291232 Mar 24 07:50 etcd-snapshot-20220324-155006.db
sftp> lpwd
Local working directory: /home/xakdwch/workstation/nfs
sftp> get etcd-snapshot-20220324-155006.db
Fetching /etcd-backup/cluster1/etcd-snapshot-20220324-155006.db to etcd-snapshot-20220324-155006.db
/etcd-backup/cluster1/etcd-snapshot-20220324-155006.db                                                                                   100%   40MB  40.3MB/s   00:01    
sftp> lls
cron-s3.yaml  cron-sftp.yaml  cron.yaml  Dockerfile  etcd-snapshot-20220324-155006.db  s3cmd-master.zip
```

从`Ceph`对象存储下载备份文件，确保本机已经安装了`s3cmd`工具：

```javascript
# s3cmd --access_key=test --secret_key=test --host="http://10.20.x.x:7480" --host-bucket="10.20.x.x:7480/%(bucket)" --no-ssl ls s3://etcd-backup/cluster1/
2022-03-24 07:46     42291232  s3://etcd-backup/cluster1/etcd-snapshot-20220324-154605.db
2022-03-24 07:48     42291232  s3://etcd-backup/cluster1/etcd-snapshot-20220324-154806.db
2022-03-24 07:50     42291232  s3://etcd-backup/cluster1/etcd-snapshot-20220324-155006.db
2022-03-24 07:52     42291232  s3://etcd-backup/cluster1/etcd-snapshot-20220324-155206.db
2022-03-24 07:54     42291232  s3://etcd-backup/cluster1/etcd-snapshot-20220324-155407.db

# s3cmd --access_key=test --secret_key=test --host="http://10.20.x.x:7480" --host-bucket="10.20.x.x:7480/%(bucket)" --no-ssl get s3://etcd-backup/cluster1/etcd-snapshot-20220324-155407.db
download: 's3://etcd-backup/cluster1/etcd-snapshot-20220324-155407.db' -> './etcd-snapshot-20220324-155407.db'  [1 of 1]
 42291232 of 42291232   100% in    4s     9.14 MB/s  done

# ls
cron-s3.yaml  cron-sftp.yaml  cron.yaml  Dockerfile  etcd-snapshot-20220324-155407.db  s3cmd-master.zip
```

## 开始恢复

以本示例为例：

**1. 停止所有的****`etcd`****和****`apiserver`****实例**

```javascript
# 停止apiserver
mv /etc/kubernetes/manifests/kube-apiserver.yaml restore/

# 停止etcd
systemctl stop etcd
```

确认所有的`etcd`、`apiserver`实例已经停止了。

**2. 恢复****`etcd`****备份数据**

在其中一个`etcd`节点上的操作如下，其它`etcd`节点操作同理。

```javascript
# 将集群现有数据备份
cp -rf /var/lib/etcd/ restore/

# 删除集群现有数据
rm -rf /var/lib/etcd/

# 之前获取的备份文件名为etcd-snapshot-20220324-155407.db，通过该文件进行etcd数据恢复
# 命令对应的参数必须根据实际情况进行填写
ETCDCTL_API=3 etcdctl snapshot restore etcd-snapshot-20220324-155407.db --data-dir=/var/lib/etcd/ --name etcd-node1 --initial-cluster etcd-node1=https://192.168.180.130:2380 --initial-cluster-token etcd-cluster-token --initial-advertise-peer-urls https://192.168.180.130:2380

# 修改目录权限
chmod 0755 /var/lib/etcd/
```

**3. 启动****`etcd`****、****`apiserver`**

```javascript
# 启动apiserver
mv restore/kube-apiserver.yaml /etc/kubernetes/manifests/

# 启动etcd
systemctl daemon-reload
systemctl restart etcd

# 验证apiserver和etcd是否都已经正常启动运行了，同时也确认其它组件实例是否正常。
# 通常也会推荐重启其它k8s组件（比如kube-scheduler, kube-controller-manager, kubelet），以确保重启后这些组件运行依赖的数据不是老数据。
```

**4. 验证服务是否正常**

最后验证`kube-system`下面的所有`pod`、`kubelet`、`etcd`服务日志没有错误信息，所有的应用是否已经启动运行了。

