

## 一、部署Ceph集群
#### 1、集群规划：
|ID  |服务器IP     |主机名  |系统版本  |
|:-: |:-:          |:-:     |:-:       |
|1   |172.16.0.111 |ceph01  |Rocky8.7  |
|2   |172.16.0.112 |ceph02  |Rocky8.7  |
|3   |172.16.0.113 |ceph03  |Rocky8.7  |
|4   |172.16.0.114 |ceph04  |Rocky8.7  |


## 二、基础环境配置（所有节点都需要）
#### 1、配置离线源：
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

#### 2、配置时间同步：
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

#### 3、离线部署docker：
```shell
[root@ceph01 ~]# cd ceph-v17.2.6/1.docker-ce
[root@ceph01 1.docker-ce]# dnf localinstall *
```

#### 4、加载ceph相关镜像：
```shell
[root@ceph01 ~]# cd ceph-v17.2.6/2.docker-images
[root@ceph01 2.docker-images]# ./load.sh 
8f690ed45de5: Loading layer [==================================================>]    225MB/225MB
0adcc251ad4a: Loading layer [==================================================>]  955.8MB/955.8MB
Loaded image: quay.io/ceph/ceph:v17.2.6-20230601
44bdf07dd71b: Loading layer [==================================================>]  570.7MB/570.7MB
Loaded image: quay.io/ceph/ceph-grafana:8.3.5
d31505fd5050: Loading layer [==================================================>]  1.459MB/1.459MB
3c7bbbb15555: Loading layer [==================================================>]  2.609MB/2.609MB
b1a388817128: Loading layer [==================================================>]  104.4MB/104.4MB
5722d45478b2: Loading layer [==================================================>]  96.33MB/96.33MB
f92428026013: Loading layer [==================================================>]  4.096kB/4.096kB
743968f98a27: Loading layer [==================================================>]  14.34kB/14.34kB
a5b14d56bc69: Loading layer [==================================================>]  28.67kB/28.67kB
bd60525c3d89: Loading layer [==================================================>]  13.31kB/13.31kB
d01dd71de373: Loading layer [==================================================>]  5.632kB/5.632kB
ad0f3e77d696: Loading layer [==================================================>]  53.25kB/53.25kB
d39b53c3618d: Loading layer [==================================================>]  2.048kB/2.048kB
85c83feccb9f: Loading layer [==================================================>]   5.12kB/5.12kB
Loaded image: quay.io/prometheus/prometheus:v2.33.4
44f62afd0479: Loading layer [==================================================>]    107MB/107MB
87cd41b1f9f8: Loading layer [==================================================>]  20.48kB/20.48kB
c096ede07531: Loading layer [==================================================>]  114.2MB/114.2MB
ccfbc5d6b2fe: Loading layer [==================================================>]   2.56kB/2.56kB
1aa420be0dba: Loading layer [==================================================>]  12.29kB/12.29kB
Loaded image: quay.io/ceph/keepalived:2.1.5
7d0ebbe3f5d2: Loading layer [==================================================>]  83.88MB/83.88MB
f8efc494ddef: Loading layer [==================================================>]  48.64kB/48.64kB
4e686e721285: Loading layer [==================================================>]  18.96MB/18.96MB
738cd65bf61e: Loading layer [==================================================>]  3.584kB/3.584kB
a0bd2f144e19: Loading layer [==================================================>]  1.536kB/1.536kB
Loaded image: quay.io/ceph/haproxy:2.3
36b45d63da70: Loading layer [==================================================>]  1.455MB/1.455MB
8d42cad20cac: Loading layer [==================================================>]   2.61MB/2.61MB
5f6d9bc8e23d: Loading layer [==================================================>]  18.23MB/18.23MB
Loaded image: quay.io/prometheus/node-exporter:v1.3.1
f1dd685eb59e: Loading layer [==================================================>]   5.88MB/5.88MB
2fb6465c7ce4: Loading layer [==================================================>]  883.2kB/883.2kB
aae8d67f7e2a: Loading layer [==================================================>]  56.23MB/56.23MB
c374f87c631e: Loading layer [==================================================>]  3.584kB/3.584kB
262e3275609f: Loading layer [==================================================>]  12.29kB/12.29kB
dc087507abf9: Loading layer [==================================================>]   5.12kB/5.12kB
b69092a6b69c: Loading layer [==================================================>]   2.56kB/2.56kB
Loaded image: grafana/loki:2.4.0
e8b689711f21: Loading layer [==================================================>]  83.86MB/83.86MB
010147bc0bbf: Loading layer [==================================================>]  3.072kB/3.072kB
5c58e4eec750: Loading layer [==================================================>]   24.8MB/24.8MB
3c1ff0b3e931: Loading layer [==================================================>]  1.242MB/1.242MB
35017d12f29d: Loading layer [==================================================>]  73.88MB/73.88MB
cab52f062b96: Loading layer [==================================================>]  3.584kB/3.584kB
Loaded image: grafana/promtail:2.4.0
03be2890f93f: Loading layer [==================================================>]  23.92MB/23.92MB
b1ea1aec9c1d: Loading layer [==================================================>]  30.95MB/30.95MB
5461fabdcb43: Loading layer [==================================================>]  3.584kB/3.584kB
ebf9953234bf: Loading layer [==================================================>]  4.096kB/4.096kB
Loaded image: quay.io/prometheus/alertmanager:v0.23.0
c3bd2e323dd4: Loading layer [==================================================>]   2.56kB/2.56kB
9f909abe8870: Loading layer [==================================================>]  10.57MB/10.57MB
bba5a257ce2f: Loading layer [==================================================>]  3.072kB/3.072kB
Loaded image: maxwo/snmp-notifier:v1.2.1
```



