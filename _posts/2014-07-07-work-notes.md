---
layout: post
title: "work notes"
description: ""
category: 
tags: []
---
{% include JB/setup %}

2014.5.4

集成测试规程分为 界面操作，正常功能测试，异常功能测试，正常功能中，如果管理员或者租户会经常调用的，需要对接口做压力测试，   
规程中可以写上测试步骤，预期结果，测试结果，是否通过，如果存在问题的，可以附上对应的ec单，由ec来跟踪，对于压力测试这种，可以直接写上测试的命令，
异常测试可以考虑一个流程未走完，启动另外一个流程，比如磁盘申请未完成，启动删除流程，也可以考虑处理过程中，对某个api直接重启或停止，测试规程上传到EC目录,
对于界面上没有完成的功能，正常操作又需要的，可以列出一个功能列表，6月份规划这些功能在界面上体现，集成测试可以通过命令行对这些基本功能进行测试


2014.5.6
{% highlight py linenos %}
import pydevd
pydevd.settrace('192.168.50.30', port=40496, stdoutToServer=True, stderrToServer=True)
{% endhighlight %}


2014.5.22

wget 192.168.88.100/sources.list.1204

wget 192.168.88.100/sources.list.1404

192.168.50.101 : bin

svn://192.168.51.200/ark/ark_v01.01.10_dev
ark_v01.r01.c00_1404_dev

5.26

[functools.partial](https://docs.python.org/2.7/library/functools.html)
property

5.27

nova network-create demo-net --bridge br100 --multi-host T --fixed-range-v4 10.0.0.0/24

PseudoFolder
StorageObject

5.28

安装swift，先装storage后装proxy<br/>
安装swift storage： 有一个分区作为存储使用，例如/dev/sdb1<br/>
提示Please input a storage device(such as /dev/sda):时，输入这个分区<br/>
安装swift proxy： 需要输入controller节点IP和root密码<br/>
提示Please input storage nodes ip:时，输入storage节点的ip，以空格隔开<br/>


6.3

1.wacp-24 91 92 99 98 96 

6.5

1.上传对象时proxy掉POST还是PUT    PUT
2.删除对象如何删
3.viso激活码C2FG9-N6J68-H8BTJ-BW3QX-RM3B3




6.6

目前的版本号，ark_v01.r01.c00.a00  ,v01表明重大结构调整，比如说版本支持ha,这个数字会发生变化，r01 ,c00都表示svn版本控制，针对不同的用户，如果需求差别较大，则会拉出不同的r分支出来，针对拉出的r分支，c00表明是dev版本，当版本开发到一定程度，需要控制开发合入新需求，并让测试针对的测试以让版本手链，就拉出c01分支，后续继续接新需求，则拉出c02分支做控制，以此类推，a00则是对svn分支发的不同的版本号，从a00-z99,一般也够用了

6.11

ntpq -p
hwclock -w


6.23

1.生成token
curl -i 'http://192.168.50.220:5000/v2.0/tokens' -X POST -H "Content-Type: application/json" -H "Accept: application/json"  -d '{"auth": {"tenantName": "admin", "passwordCredentials": {"username": "admin", "password": "123456"}}}'

curl -i -X GET -H "X-Auth-Token: $token" "http://192.168.50.220:8776/v1/55531793bd004e5284c8f77005b53520/volumes"


6.25

cinder关于type的配置

enabled_backends=lvm1,nfs1<br/>
[lvm1]<br/>
volume_driver=cinder.volume.drivers.lvm.LVMISCSIDriver<br/>
volume_backend_name=LVM_iSCSI<br/>
[nfs1]<br/>
nfs_shares_config=${PATH_TO_YOUR_SHARES_FILE}<br/>
volume_driver=cinder.volume.drivers.nfs.NfsDriver<br/>
volume_backend_name=NFS<br/>


6.30

在RingBuilder类内使用min_part_hours来限制在规定时间内，同一个partition不会被移动两次。<br/>
构建相关的builder文件：
`swift-ring-builder <builder_file> create <part_power> <replicas> <min_part_hours>`
 
添加node到builder文件：
`swift-ring-builder <builder_file> add  z<zone>-<ip>:<port>/<device_name>_<meta> <weight>`


7.3

HA 版本重启服务crm_mon 查看资源ID 
`crm resource restart <resource-id>`


7.5

1.apscheduler给Job传参的方法：<br>
{% highlight py linenos %}
from apscheduler.scheduler import Scheduler

sched = Scheduler(daemonic=False)
def job(text):
    print text

sched.add_cron_job(job, hour=17, minute=36, args=['Hello World'])
sched.start()
{% endhighlight%}

