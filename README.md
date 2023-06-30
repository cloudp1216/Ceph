

## 一、Ceph集群规划
|ID  |服务器IP     |主机名  |系统版本  |
|:-: |:-:          |:-:     |:-:       |
|1   |172.16.0.111 |ceph01  |Rocky8.7  |
|2   |172.16.0.112 |ceph02  |Rocky8.7  |
|3   |172.16.0.113 |ceph03  |Rocky8.7  |
|4   |172.16.0.114 |ceph04  |Rocky8.7  |


## 二、基础环境配置（除了免密登录其它节点都需要配置）
#### 1、调整hosts：
```shell
[root@ceph01 ~]# vi /etc/hosts
...
172.16.0.111 ceph01 registry
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

#### 6、调整docker配置文件，指定registry：
```shell
[root@ceph01 ~]# vi /etc/docker/daemon.json
{
  "insecure-registries": ["registry:5000"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```


## 三、部署registry（仅在ceph01部署）
#### 1、导入registry镜像：
```shell
[root@ceph01 ~]# cd ceph-v17.2.6/2.registry/
[root@ceph01 2.registry]# ./load.sh
```

#### 2、启动私有registry：
```shell
[root@ceph01 ~]# docker run -d --name registry --restart always -p 5000:5000 -v /var/lib/registry:/var/lib/registry registry:2.8.2
```


## 四、导入ceph镜像
#### 1、导入ceph相关镜像（所有节点都需要）：
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

#### 2、推送ceph镜像到私有仓库：
```shell
[root@ceph01 ~]# docker tag quay.io/ceph/ceph:v17 registry:5000/ceph:v17
[root@ceph01 ~]# docker push registry:5000/ceph:v17
```


## 五、安装cephadm、ceph-common（仅管理节点）
#### 1、安装cephadm
```shell
[root@ceph01 ~]# cp ceph-v17.2.6/4.cephadm/cephadm /usr/sbin
```

#### 2、安装依赖：
```shell
[root@ceph01 ~]# cd ceph-v17.2.6/5.ceph/1.epel-dependent/
[root@ceph01 1.epel-dependent]# dnf localinstall *
```

#### 3、安装ceph-common：
```shell
[root@ceph01 ~]# cd ceph-v17.2.6/5.ceph/2.ceph/
[root@ceph01 2.ceph]# dnf localinstall *
```


## 六、初始化集群
#### 1、初始化集群：
```shell
[root@ceph01 ~]# cephadm --image registry:5000/ceph:v17 bootstrap --mon-ip 172.16.0.111 --skip-pull
Verifying podman|docker is present...
Verifying lvm2 is present...
Verifying time synchronization is in place...
Unit chronyd.service is enabled and running
Repeating the final host check...
docker (/usr/bin/docker) is present
systemctl is present
lvcreate is present
Unit chronyd.service is enabled and running
Host looks OK
Cluster fsid: f0a7abc8-16ec-11ee-a634-5254009a2a54
Verifying IP 172.16.0.111 port 3300 ...
Verifying IP 172.16.0.111 port 6789 ...
Mon IP `172.16.0.111` is in CIDR network `172.16.0.0/24`
Mon IP `172.16.0.111` is in CIDR network `172.16.0.0/24`
Internal network (--cluster-network) has not been provided, OSD replication will default to the public_network
Ceph version: ceph version 17.2.6 (d7ff0d10654d2280e08f1ab989c7cdf3064446a5) quincy (stable)
Extracting ceph user uid/gid from container image...
Creating initial keys...
Creating initial monmap...
Creating mon...
Waiting for mon to start...
Waiting for mon...
mon is available
Assimilating anything we can from ceph.conf...
Generating new minimal ceph.conf...
Restarting the monitor...
Setting mon public_network to 172.16.0.0/24
Wrote config to /etc/ceph/ceph.conf
Wrote keyring to /etc/ceph/ceph.client.admin.keyring
Creating mgr...
Verifying port 9283 ...
Waiting for mgr to start...
Waiting for mgr...
mgr not available, waiting (1/15)...
mgr not available, waiting (2/15)...
mgr not available, waiting (3/15)...
mgr not available, waiting (4/15)...
mgr is available
Enabling cephadm module...
Waiting for the mgr to restart...
Waiting for mgr epoch 5...
mgr epoch 5 is available
Setting orchestrator backend to cephadm...
Generating ssh key...
Wrote public SSH key to /etc/ceph/ceph.pub
Adding key to root@localhost authorized_keys...
Adding host ceph01...
Deploying mon service with default placement...
Deploying mgr service with default placement...
Deploying crash service with default placement...
Deploying ceph-exporter service with default placement...
Deploying prometheus service with default placement...
Deploying grafana service with default placement...
Deploying node-exporter service with default placement...
Deploying alertmanager service with default placement...
Enabling the dashboard module...
Waiting for the mgr to restart...
Waiting for mgr epoch 9...
mgr epoch 9 is available
Generating a dashboard self-signed certificate...
Creating initial admin user...
Fetching dashboard port number...
Ceph Dashboard is now available at:

         URL: https://ceph01:8443/
        User: admin
    Password: ue68ni7ifv

Enabling client.admin keyring and conf on hosts with "admin" label
Saving cluster configuration to /var/lib/ceph/f0a7abc8-16ec-11ee-a634-5254009a2a54/config directory
Enabling autotune for osd_memory_target
You can access the Ceph CLI as following in case of multi-cluster or non-default config:

    sudo /usr/sbin/cephadm shell --fsid f0a7abc8-16ec-11ee-a634-5254009a2a54 -c /etc/ceph/ceph.conf -k /etc/ceph/ceph.client.admin.keyring

Or, if you are only running a single cluster on this host:

    sudo /usr/sbin/cephadm shell 

Please consider enabling telemetry to help improve Ceph:

    ceph telemetry on

For more information see:

    https://docs.ceph.com/docs/master/mgr/telemetry/

Bootstrap complete.
```

#### 2、复制集群公钥到其它节点：
```shell
[root@ceph01 ~]# ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph02
[root@ceph01 ~]# ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph03
[root@ceph01 ~]# ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph04
```

#### 3、ceph01打上管理员标签：
```shell
[root@ceph01 ~]# ceph orch host label add ceph01 _admin
Added label _admin to host ceph01
[root@ceph01 ~]# ceph orch host ls
HOST    ADDR          LABELS  STATUS  
ceph01  172.16.0.111  _admin          
1 hosts in cluster
```

#### 4、添加节点ceph02、ceph03、ceph04到集群
```shell
[root@ceph01 ~]# ceph orch host add ceph02 172.16.0.112
Added host 'ceph02' with addr '172.16.0.112'
[root@ceph01 ~]# ceph orch host add ceph03 172.16.0.113
Added host 'ceph03' with addr '172.16.0.113'
[root@ceph01 ~]# ceph orch host add ceph04 172.16.0.114
Added host 'ceph04' with addr '172.16.0.114'
```

#### 5、添加mon
```shell
[root@ceph01 ~]# ceph orch apply mon "ceph01,ceph02,ceph03"
Scheduled mon update...
```

#### 6、添加mgr
```shell
[root@ceph01 ~]# ceph orch apply mgr "ceph01,ceph02,ceph03"
Scheduled mgr update...
```

#### 7、查看集群进程
```shell
[root@ceph01 ~]# ceph orch ps
NAME                  HOST    PORTS        STATUS          REFRESHED   AGE  MEM USE  MEM LIM  VERSION  IMAGE ID      CONTAINER ID  
alertmanager.ceph01   ceph01  *:9093,9094  running (102s)    24s ago   38m    15.3M        -  0.23.0   ba2b418f427c  0069c58b5e83  
ceph-exporter.ceph01  ceph01               running (38m)     24s ago   38m    6483k        -  17.2.6   52bedc025a3c  e8bf7bcfd3d5  
ceph-exporter.ceph02  ceph02               running (9m)       7m ago    9m    6055k        -  17.2.6   52bedc025a3c  a3dbed25490b  
ceph-exporter.ceph03  ceph03               running (8m)     104s ago    8m    6251k        -  17.2.6   52bedc025a3c  1ca18243f16f  
ceph-exporter.ceph04  ceph04               running (7m)       2m ago    7m    6175k        -  17.2.6   52bedc025a3c  86839b6f9128  
crash.ceph01          ceph01               running (38m)     24s ago   38m    7063k        -  17.2.6   52bedc025a3c  d59730c8370f  
crash.ceph02          ceph02               running (9m)       7m ago    9m    7067k        -  17.2.6   52bedc025a3c  33db93481b22  
crash.ceph03          ceph03               running (8m)     104s ago    8m    7084k        -  17.2.6   52bedc025a3c  d29a3e4615d6  
crash.ceph04          ceph04               running (7m)       2m ago    7m    7071k        -  17.2.6   52bedc025a3c  dc05c71fd6b3  
grafana.ceph01        ceph01  *:3000       running (37m)     24s ago   38m    55.8M        -  8.3.5    dad864ee21e9  d86dc5d122b0  
mgr.ceph01.gwcwvz     ceph01  *:9283       running (39m)     24s ago   39m     474M        -  17.2.6   52bedc025a3c  42bdc30cf72a  
mgr.ceph02.ztsjzq     ceph02  *:8443,9283  running (9m)       7m ago    9m     407M        -  17.2.6   52bedc025a3c  273a8c48017f  
mgr.ceph03.otsafz     ceph03  *:8443,9283  running (111s)   104s ago  111s     141M        -  17.2.6   52bedc025a3c  15f48d26a681  
mon.ceph01            ceph01               running (39m)     24s ago   39m    50.4M    2048M  17.2.6   52bedc025a3c  cd3f84b9495f  
mon.ceph02            ceph02               running (8m)       7m ago    8m    35.5M    2048M  17.2.6   52bedc025a3c  7d844642fd99  
mon.ceph03            ceph03               running (8m)     104s ago    8m    36.4M    2048M  17.2.6   52bedc025a3c  643d3b4a604a  
node-exporter.ceph01  ceph01  *:9100       running (38m)     24s ago   38m    23.1M        -  1.3.1    1dbe0e931976  cce5b64ed840  
node-exporter.ceph02  ceph02  *:9100       running (8m)       7m ago    8m    16.1M        -  1.3.1    1dbe0e931976  c7408086dd71  
node-exporter.ceph03  ceph03  *:9100       running (8m)     104s ago    8m    11.3M        -  1.3.1    1dbe0e931976  74d30088eff4  
node-exporter.ceph04  ceph04  *:9100       running (7m)       2m ago    7m    16.3M        -  1.3.1    1dbe0e931976  de06157fd16d  
prometheus.ceph01     ceph01  *:9095       running (100s)    24s ago   37m    49.3M        -  2.33.4   514e6a882f6e  f2af554421d6  
```



