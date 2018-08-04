---
title: 为AWS AutoScaling持续提供最新映像
date: 2018-07-18 18:22:26
tags:
- AWS
- AutoScaling
categories:
- DevOps
---

## 背景

通过之前的踩坑与学习，目前我对 AWS Auto Scaling 已经有了一定程度的了解。那么对完成工作需求也就有了更高的标准。  
**新的问题是：**随着业务功能的更新，如何保证 Auto Scaling 可以扩容出与线上保持版本一致的 EC2 实例？  
这就要求我们，在业务更新发版后，及时的去更新 Auto Scaling 所配置的映像（Image）。

<!--more-->

## 基本思路

需求清晰了，就很容易得出实现方法。只要在业务更新发版后，重新制作映像，然后关联在 Auto Scaling Group 上就好了。  
映像与 Auto Scaling Group 的关联可以通过两种方式：

1. 启动配置（Launch Configuration）
2. 启动模板（Launch Template）

那么使用哪种方式更好呢？稍微分析一下就知道了：  
**启动配置**创建后不可修改，只能创建新的启动配置与最新的映像关联，然后再修改 Auto Scaling Group 与新创建的启动配置关联。替换掉的启动配置不好管理。  
**启动模板**的优势在于有**版本**的概念。可以在上一版的基础上，替换最新制作的映像生成启动模板的新版本。不需要再更新 Auto Scaling Group 的相关属性。通过版本这个概念也可以很好的管理历史的映像。

整个流程如下：

1. 手动创建初版的启动模板（Launch Template）。
2. 将 Auto Scaling Group 与启动模板关联，注意启动模板的版本设置为**最新**。
3. Jenkins构建发版后触发脚本。
4. 脚本运行
    1. 调用 AWS API 为机器生成映像。
    2. 调用 AWS API 使用新生成的映像创建启动模板的一个新版本。
5. 完成。之后新扩容出的机器将使用最新的映像。


## 注意事项

1. Auto Scaling Group 与启动模板关联，要设置的关联模板版本为**最新**。
![config-asg.png](./config-asg.png)

2. 创建映像从开始到完成需要一段时间，在映像创建完成到达可用状态之前，Auto Scaling Group 将会扩容失败。映像可用后如达到扩容条件，将再次扩容。
![auto-scaling-failed.png](./auto-scaling-failed.png)


## Demo Code

```python
# coding:utf-8
import boto3
import logging
import datetime
import argparse

logging.basicConfig(
    format='%(levelname)s %(asctime)s\n%(message)s',
    datefmt='%Y-%m-%d %H:%M:%S',
    filename='./auto_ami.log',
    level=logging.INFO,
    filemode='a'
)


def get_all_amis(client):
    """
    获取本账户下的所以映像
    :param client:
    :return:
    """
    params = {'Owners': ['909440319734']}
    result = client.describe_images(**params)
    return result['Images']


def create_image(client, instanceid, name):
    """
    为指定的实例创建映像
    :param client: boto3 ec2 client
    :param instanceid: EC2 InstanceId
    :param name: 映像名称
    :return: ImageId
    """
    params = {
        'InstanceId': instanceid,
        'NoReboot': True,
        'Name': name,
        'Description': 'Auto created for auto scaling at {time}.'.format(time=datetime.datetime.now().strftime('%Y%m%d%H%M%S'))
    }
    logging.info('try to create image({name}) for {instance}.'.format(name=name, instance=instanceid))
    try:
        result = client.create_image(**params)
        imageid = result['ImageId']
        logging.info('start create image({name}) for {instance}.'.format(name=name, instance=instanceid))
        return imageid
    except Exception as e:
        logging.exception('failed to create image({name}) for {instance}.'.format(name=name, instance=instanceid))
        return None


def create_launch_template_version(client, templateid, ami):
    """
    使用指定映像创建指定启动模板的新版本
    :param client: boto3 ec2 client
    :param templateid: Launch TemplateId
    :param ami: ImageId
    :return: Launch Template Version Number
    """
    try:
        result = client.describe_launch_templates(LaunchTemplateIds=[templateid])
        template = result['LaunchTemplates'][0]
    except Exception as e:
        logging.exception(str(e))
        logging.error('can not found launch template({templateid}) to create new version with ami(imageid).'.format(templateid=templateid, imageid=ami))
        return

    logging.info('try to create launch template({template}) new version with image({image}).'.format(template=templateid, image=ami))
    try:
        latest_version = str(template['LatestVersionNumber'])
        result = client.create_launch_template_version(
            LaunchTemplateId=templateid,
            SourceVersion=latest_version,
            VersionDescription='Auto created for autoscaling at {time}'.format(time=datetime.datetime.now().strftime('%Y%m%d%H%M%S')),
            LaunchTemplateData={'ImageId': ami}
        )
        return result['LaunchTemplateVersion']['VersionNumber']
    except Exception as e:
        logging.exception(str(e))
        logging.error('create launch template({templateid}) new version with ami(imageid) failed.'.format(templateid=templateid, imageid=ami))
        return


def main(args, config):
    region, app, layer, env, group = args.region, args.app, args.layer, args.env, args.group

    app_instance_map = config['app_instance_map']
    aws_ak = config['aws_ak']
    aws_sk = config['aws_sk']

    client = boto3.client('ec2', aws_access_key_id=aws_ak, aws_secret_access_key=aws_sk, region_name='us-east-1')

    info = app_instance_map[region][app][layer][env][group]
    # 创建镜像
    image_name = '{region}-{app}-{layer}-{env}-{group}-{time}'.format(region=region, app=app, layer=layer, env=env, group=group, time=datetime.datetime.now().strftime('%Y%m%d%H%M%S'))
    imageid = create_image(client=client, instanceid=info['instance'], name=image_name)
    if not imageid:
        logging.error('can not get imageid. program exit.')
        return
    logging.info('create instance({instance})\'s image({image}) successfully.'.format(instance=info['instance'], image=imageid))

    # 创建启动模板新版本
    version_number = create_launch_template_version(client=client, templateid=info['launch_template'], ami=imageid)
    if not version_number:
        logging.error('can not get launch template new version number. program exit.')
        return
    logging.info('create launch template({template}) new version({version}) successfully.'.format(template=info['launch_template'], version=version_number))

    logging.info('SUCCESS')



if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('--region', type=str, choices=['vg', 'bj'], help='Please specify the region')
    parser.add_argument('--app', type=str, help='Please specify the app')
    parser.add_argument('--layer', type=str, choices=['web', 'app'], help='Please specify the layer')
    parser.add_argument('--env', type=str, choices=['prod', 'release'], help='Please specify the env')
    parser.add_argument('--group', type=str, choices=['a', 'b'], help='Please specify the group')
    args = parser.parse_args()

    config = {
        'aws_ak': '你的 AWS AccessKey',
        'aws_sk': '你的 AWS AccessSecret',
        'app_instance_map': {
        # 根据具体的业务逻辑按项目、环境来分别配置 InstanceId 和 LaunchTemplateId
            'vg': {
                'xxx': {
                    'app': {
                        'release': {
                            'a': {
                                'instance': 'i-xxxxxxxxxxxxxxxxx',
                                'launch_template': 'lt-xxxxxxxxxxxxxxxxx'
                            },
                            'b': {
                                'instance': 'i-xxxxxxxxxxxxxxxxx',
                                'launch_template': 'lt-xxxxxxxxxxxxxxxxx'
                            }
                        },
                    },
                },
            }
        }
    }
    main(args, config)

```
