

## 一、部署Ceph集群
#### 1、集群规划：
|ID  |服务器IP     |主机名  |系统版本  |
|:-: |:-:          |:-:     |:-:       |
|1   |172.16.0.111 |ceph01  |Rocky8.7  |
|2   |172.16.0.112 |ceph02  |Rocky8.7  |
|3   |172.16.0.113 |ceph03  |Rocky8.7  |
|4   |172.16.0.114 |ceph04  |Rocky8.7  |


## 二、基础环境配置（所有节点都需要配置）
#### 1、调整hosts：
```shell
[root@ceph01 ~]# vi /etc/hosts
...
172.16.0.111 ceph01
172.16.0.112 ceph02
172.16.0.113 ceph03
172.16.0.114 ceph04

```

#### 2、配置ceph01到其它节点免密：
```shell
[root@ceph01 ~]# ssh-keygen
[root@ceph01 ~]# ssh-copy-id root@ceph02
[root@ceph01 ~]# ssh-copy-id root@ceph03
[root@ceph01 ~]# ssh-copy-id root@ceph04
```

#### 3、配置离线源：
```shell
[root@ceph01 ~]# vi /etc/yum.repos.d/Local.repo 


[baseos]
name=Rocky Linux $releasever - BaseOS
baseurl=file:///media/BaseOS/
enabled=1
countme=1
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-rockyofficial


[appstream]
name=Rocky Linux $releasever - AppStream
baseurl=file:///media/AppStream/
enabled=1
countme=1
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-rockyofficial


```
```shell
[root@ceph01 ~]# mount -o loop Rocky-8.7-x86_64-dvd1.iso /media/
mount: /media: WARNING: device write-protected, mounted read-only.
[root@ceph01 ~]# dnf makecache
```

#### 4、配置时间同步：
```shell
[root@ceph01 ~]# dnf install chrony
[root@ceph01 ~]# vi /etc/chrony.conf
...
pool ntp.speech.local iburst
...
[root@ceph01 ~]# systemctl restart chronyd
[root@ceph01 ~]# chronyc -a makestep
200 OK
```

#### 5、离线部署docker：
```shell
[root@ceph01 ~]# cd ceph-v17.2.6/1.docker-ce
[root@ceph01 1.docker-ce]# dnf localinstall *
```

#### 6、加载ceph相关镜像：
```shell
[root@ceph01 ~]# cd ceph-v17.2.6/2.docker-images
[root@ceph01 2.docker-images]# ./load.sh 
[root@ceph01 2.docker-images]# docker images
REPOSITORY                         TAG                IMAGE ID       CREATED         SIZE
quay.io/ceph/ceph                  v17                52bedc025a3c   3 weeks ago     1.16GB
quay.io/ceph/ceph                  v17.2.6-20230601   52bedc025a3c   3 weeks ago     1.16GB
quay.io/ceph/ceph-grafana          8.3.5              dad864ee21e9   14 months ago   558MB
quay.io/prometheus/prometheus      v2.33.4            514e6a882f6e   16 months ago   204MB
quay.io/ceph/keepalived            2.1.5              9f7bdb4a87fd   16 months ago   214MB
quay.io/ceph/haproxy               2.3                e85424b0d443   17 months ago   99.3MB
quay.io/prometheus/node-exporter   v1.3.1             1dbe0e931976   18 months ago   20.9MB
grafana/loki                       2.4.0              24d3d94c71c7   19 months ago   62.5MB
grafana/promtail                   2.4.0              f568284f5b06   19 months ago   179MB
quay.io/prometheus/alertmanager    v0.23.0            ba2b418f427c   22 months ago   57.5MB
maxwo/snmp-notifier                v1.2.1             7ca9dd8b3f09   22 months ago   13.2MB
```


