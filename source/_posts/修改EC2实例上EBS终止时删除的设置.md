---
title: 修改EC2实例上EBS终止时删除的设置
date: 2018-10-23 11:23:04
permalink: modify-delete-on-termination-config-for-ebs-on-ec2-instance
tags:
- AWS
categories:
- DevOps
---

前段时间把很多项目都上了 Auto Scaling 功能，这两天发现有些 EC2 实例缩减后留下了 EBS 没有释放。这是因为实例的块存设备(Block devices)没有设置**终止时释放(DeleteOnTermination)**。  
在启动新实例或创建 AMI 映像时可设置**终止时释放(DeleteOnTermination)**，但已启动的实例该怎么办呢？就此问题我求助了 AWS 客户支持，在这里把解决方案分享出来。
<!--more-->

## 背景

Auto Scaling Group 缩减实例时没有释放掉关联的 EBS，因为实例上 EBS **终止时释放(DeleteOnTermination)**的选项设置为 False。无法在 AWS Console 上修改已启动的实例的此选项。


## 需求

修改已启动的实例上 EBS **终止时释放(DeleteOnTermination)**的选项。


## 方案

通过AWS CLI来修改。

在本地创建一个文件 mapping.json，内容如下（DeviceName 需按实际情况修改）：
```json
[
    {
        "DeviceName": "/dev/sda1",
        "Ebs": {
            "DeleteOnTermination": true
        }
    },
    {
        "DeviceName": "/dev/sdb",
        "Ebs": {
            "DeleteOnTermination": true
        }
    }
]
```

执行如下 AWS CLI 命令（需要 ModifyInstanceAttribute 权限）：
```shell
 aws ec2 modify-instance-attribute --instance-id i-0679a59xxxxxxxxxx --block-device-mappings file://mapping.json
```

## 总结

开始没找到相应的 API，后来发现其实 boto3 等 AWS SDK 也是可以的，例如 boto3 中 [modify_instance_attribute](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ec2.html#EC2.Client.modify_instance_attribute) 函数。

没事的时候要多熟悉熟悉文档，求助可以更快的解决问题，但不能有依赖的心理！


