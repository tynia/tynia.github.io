---
layout: post
title: 在CentOS7上手动部署ceph集群
subtitle: 踩坑札记
date:   2017-06-22 17:01:39
categories: ceph
tag: ceph 集群部署
---

## 准备
ceph的（离线）手动安装很繁琐，最好是能有本地镜像源或者能访问到公共镜像源。不再敖述。
安装完成后，所有ceph的工具，都能在控制台访问到，例如：
```
# whereis ceph
ceph: /usr/bin/ceph /usr/lib64/ceph /etc/ceph /usr/libexec/ceph /usr/share/ceph /usr/share/man/man8/ceph.8.gz
# ls /usr/bin/ | grep ceph
…
```
---
## 环境：
ceph 集群部署：
一个mon，2个osd，再扩展到3个osd；
一个机器上只有一个节点，要么是mon，要么是osd；
各个节点之间，都能使用域名能访问到。
```
# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.1.151 ceph-osd-1
192.168.1.152 ceph-osd-2
192.168.1.153 ceph-osd-3
192.168.1.154 monitor
```

各个机器及环境：
```
角色      host            ip            os
mon     monitor      192.168.1.154   centos 7
osd-0   ceph-osd-1   192.168.1.151   centos 7
osd-1   ceph-osd-2   192.168.1.152   centos 7
osd-2   ceph-osd-3   192.168.1.153   centos 7
```

---
## 部署
### 部署mon
1. 登录到 ceph-monitor
2. 首先，要生成集群的唯一标识：
```
# uuidgen
07be6a13-e992-40fc-9941-50d19fd4450f
```
> 一个集群必须有一个唯一的标识，这个将在```ceph.conf```中的```fsid```体现。

3. 生成监视器密钥（放在当前路径）：
```
# ceph-authtool --create-keyring ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
```

4. 生成管理员密钥，放在指定的路径：
```
# ceph-authtool --create-key /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --set-uid=0 --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow'
```

5. 把管理员密钥，导入到监视器密钥文件中：
```
# ceph-authtool ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
```

6. 生成监视器视图，保存为指定名字：
```
# monmaptool --create --add ceph-monitor 192.168.1.154 --fsid 07be6a13-e992-40fc-9941-50d19fd4450f monmap.dat
```

7. 创建监视数据目录：
```
# mkdir /var/lib/ceph/mon/ceph-monitor
```

8. 用生成的监视器视图，初始化监视器：
```
# ceph-mon --mkfs -i monitor --monmap ./monmap.bin --keyring ./ceph.mon.keyring
```

9. 创建 ceph.conf：
```
# vi /etc/ceph/ceph.conf
```

  写入以下内容：（根据自己的环境编写）
  ```
  [global]
  fsid = 07be6a13-e992-40fc-9941-50d19fd4450f # uuidgen 生成的唯一标识
  mon_initial_members = monitor
  mon_host = 192.168.1.154
  auth_supported = cephx
  auth_cluster_required = cephx
  auth_client_required = cephx
  auth_service_required = cephx
  osd_pool_default_crush_rule = 2
  osd_pool_default_size = 2
  osd_pool_default_pg_num = 2048
  osd_pool_default_pgp_num = 2048
  osd_crush_chooseleaf_type = 0

  [mon.monitor]
  host = monitor
  mon_data = /var/lib/ceph/mon/ceph-monitor
  mon_addr = 192.168.1.154:6789

  [osd.0]
  host = ceph-osd-1
  osd_data = /var/lib/ceph/osd/ceph-0

  [osd.1]
  host = ceph-osd-2
  osd_data = /var/lib/ceph/osd/ceph-1

  [osd.2]
  host = ceph-osd-3
  osd_data = /var/lib/ceph/osd/ceph-2
  ```

10. 创建一个空白文件，表示监视器已经创建完成，待启动：
```
# touch /var/lib/ceph/mon/ceph-monitor/done
```

11. 启动monitor节点：
```
# /etc/init.d/ceph start mon.monitor
```

12. 设置开机自启动：
```
# touch /var/lib/ceph/mon/ceph-monitor/sysvinit
```

13. 验证ceph启动：
```
# ceph osd lspools
0 rbd,
# ceph -s
    cluster 07be6a13-e992-40fc-9941-50d19fd4450f
     health HEALTH_ERR
            no osds
     monmap e1: 1 mons at {liuzhen-ceph-monitor=192.168.1.154:6789/0}
            election epoch 1, quorum 0 liuzhen-ceph-monitor
     osdmap e10: 0 osds: 0 up, 0 in
      pgmap v11: 0 pgs, 0 pools, 0 bytes data, 0 objects
            0 kB used, 0 kB / 0 kB avail
```
  发现 集群的健康状态是**HEALTH_ERR**。不要惊慌，这是因为当前集群中只有一个监视节点，还没有其它osd节点的缘故。当加入了osd节点之后，集群状态就会有所改变。

  接下来，我们准备安装osd节点。首先是要把集群的mon节点的信息告诉osd节点。

14.  把/etc/ceph/ceph.conf发给其它的osd节点（因为fsid是集群id，该集群下的osd的配置中，集群的fsid必须一致，体现在ceph.conf中[global]配置段内）。
  另外，如果要在osd上操作mon的话，需要把mon的管理员身份信息——keyring发给各个osd节点：
  ```
  # scp /etc/ceph/ceph.client.admin.keyring root@ceph-osd-1:/etc/ceph/
  # scp /etc/ceph/ceph.client.admin.keyring root@ceph-osd-2:/etc/ceph/
  # scp /etc/ceph/ceph.client.admin.keyring root@ceph-osd-3:/etc/ceph/
  ```

  > 注意，这个操作的目的是为了方便，能在osd节点也能以管理员身份操作mon节点添加osd，并不推荐。如果没有这一步，那么在创建osd的时候，只能在mon的节点上创建osd。

---
## 部署osd
1. 登录到```ceph-osd-1（osd节点）```

2. 先查看一下该节点机器的挂载盘：
```
# lsblk -l
NAME MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sr0   11:0    1  422K  0 rom
vda  253:0    0   20G  0 disk
vda1 253:1    0   20G  0 part /
vdb  253:16   0   50G  0 disk
vdc  253:32   0   50G  0 disk
vdd  253:48   0   50G  0 disk
vde  253:64   0   50G  0 disk
```
  ceph-osd-1上有以上几个磁盘，现在要把vdb当做osd的数据存储盘。

3. 把vdb格式化成xfs格式：
```
# mkfs -t xfs -f /dev/vdb
meta-data=/dev/vdb               isize=256    agcount=4, agsize=3276800 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=0        finobt=0
data     =                       bsize=4096   blocks=13107200, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=0
log      =internal log           bsize=4096   blocks=6400, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```
4. 创建osd的数据目录：
```
# mkdir /var/lib/ceph/osd/ceph-0
```

5. 把vdb挂到osd的数据目录：
```
# mount -t xfs -o rw,nodev,noexec,noatime,nodiratime,attr2,discard,inode64,logbsize=256k,noquota /dev/vdb /var/lib/ceph/osd/ceph-0
```

6. 创建一个osd节点：
```
ceph osd create [--cluster cluster-name]
0
```

  默认是在集群名字为```ceph```的集群下创建节点，如果不是在默认集群下创建osd节点，则需要加上```--cluster```来指定集群。
  创建完成后，会生成一个节点，当前是默认集群```ceph```下的第一个osd节点，**osd-id**给出的是**0**。牢记，在初始化osd节点的时候会使用到。

7. 获取/dev/vdb的uuid：
```
# lsblk -f
NAME   FSTYPE  LABEL    UUID                                 MOUNTPOINT
sr0    iso9660 config-2 2017-06-09-17-14-20-00               
vda                                                          
└─vda1 xfs              3ef2bb69-f3dc-436e-b992-7e90057257f9 /
vdb    xfs              195e4405-a03f-46ad-88ca-3fac00f4f0f8 /var/lib/ceph/osd/ceph-0
vdc                                                          
vdd                                                          
vde
```
可以看到，挂载磁盘vdb的**uuid**是 **195e4405-a03f-46ad-88ca-3fac00f4f0f8**。

8. 初始化osd节点的数据目录：
```
# ceph-osd -i {osd-id} --mkfs --mkkey --osd-uuid [{uuid}]
2017-06-22 10:41:12.266321 7fd2b87b1880 -1 journal FileJournal::_open: disabling aio for non-block journal.  Use journal_force_aio to 
2017-06-22 10:41:12.376429 7fd2b87b1880 -1 journal FileJournal::_open: disabling aio for non-block journal.  Use journal_force_aio to 
2017-06-22 10:41:12.376941 7fd2b87b1880 -1 filestore(/var/lib/ceph/osd/ceph-1) could not find -1/23c2fcde/osd_superblock/0 in index: (
2017-06-22 10:41:12.465759 7fd2b87b1880 -1 created object store /var/lib/ceph/osd/ceph-1 journal /var/lib/ceph/osd/ceph-1/journal for 
2017-06-22 10:41:12.465808 7fd2b87b1880 -1 auth: error reading file: /var/lib/ceph/osd/ceph-1/keyring: can't open /var/lib/ceph/osd/ce
2017-06-22 10:41:12.465985 7fd2b87b1880 -1 created new key in keyring /var/lib/ceph/osd/ceph-1/keyring
```
  其中```{osd-id}```即为创建osd节点时候生成的id，```{uuid}```是挂在盘vdb的uuid。

9. 注册osd的密钥：
```
# ceph auth add osd.{osd-id} osd 'allow *' mon 'allow rwx' -i /var/lib/ceph/osd/ceph-0/keyring
added key for osd.0
```
> 注意，如果该osd不是在默认的```ceph```集群下，则要显式指定集群。

10. 把该osd节点，添加到crush视图：
```
# ceph osd crush add-bucket ceph-osd-1 host
added bucket ceph-osd-1 type host to crush map
```

11. 把该节点所属的域名，添加到default根下
```
# ceph osd crush move ceph-osd-1 root=default
moved item id -2 name 'ceph-osd-1' to location {root=default} in crush map
```

12. 给osd节点分配权重，并添加到指定的host下
```
# ceph osd crush add osd.1 1.0 host=ceph-osd-1
add item id 0 name 'osd.0' weight 1 at location {host=ceph-osd-1} to crush map
```

13. 查询一下ceph的状态：
```
# ceph -s
    cluster 07be6a13-e992-40fc-9941-50d19fd4450f
     health HEALTH_WARN
            too few PGs per OSD (0 < min 30)
            1/1 in osds are down
     monmap e1: 1 mons at {monitor=192.168.1.154:6789/0}
            election epoch 1, quorum 0 liuzhen-ceph-monitor
     osdmap e15: 1 osds: 0 up, 1 in
      pgmap v16: 0 pgs, 0 pools, 0 bytes data, 0 objects
            0 kB used, 0 kB / 0 kB avail
```
> 注意，如果输出中，osdmap段给的是 1osds：0 up，0 in，则需要显示标明，osd节点的状态是in状态。执行```ceph osd in {osd-id}```。{osd-id}是当前的osd的id，即**0**。

14. 启动osd节点
```
# /etc/init.d/ceph start osd.0
=== osd.0 === 
create-or-move updated item name 'osd.0' weight 0.05 at location {host=ceph-osd-1,root=default} to crush map
Starting Ceph osd.0 on ceph-osd-1...
Running as unit ceph-osd.0.1498099035.882680235.service.
```

15. 查询当前ceph集群的状态
```
# ceph osd tree
ID WEIGHT  TYPE NAME                   UP/DOWN REWEIGHT PRIMARY-AFFINITY 
-1 2.00000 root default                                                  
-2 1.00000     host liuzhen-ceph-osd-1                                   
 0 1.00000         osd.0                    up  1.00000          1.00000
```
可见，osd已经顺利运行起来，可以开始接受数据了。

---
## 接下来
用同样的方式，在```ceph-osd-2```上创建第二个osd，在```ceph-osd-3```上创建第三个osd。
方法相同，不在重复描述。

---
## 结束
本文通过创建一个mon节点和一个osd节点，在centos 7上**手动**搭建一个简单的ceph集群。依照如此，可以扩展出多个**mon**节点和多个**osd**节点的集群。

ceph是一个庞大的系统，官网的描述不太准确，和其名气与气质有较大的偏差。这也导致我们团队在学习的时候，在安装部分就遇到困难。

希望ceph社区不要光只关注这功能的实现，在文档方面，也能达到其性能水平。

## 参考
1. [ceph官方文档：手动部署](http://docs.ceph.com/docs/master/install/manual-deployment/)
