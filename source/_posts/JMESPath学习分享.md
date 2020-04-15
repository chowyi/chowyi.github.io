---
title: JMESPath学习分享
date: 2020-04-14 22:17:05
permalink: jmespath-learning-notes
tags: JMESPath
categories: DevOps
---

在学习 AWS CLI 文档时偶然了解到 JMESPath，之前从没接触到类似的技术，赶紧学习了一番，顿觉收获颇丰，在此记录分享。
<!--more-->

## 什么是JMESPath

先贴上官网地址：<https://jmespath.org/>.  

JMESPath 读作 “James Path”，因为它的作者名字叫 [James Saryerwinnie](https://github.com/jamesls)。  

官网上的第一句话就是对 JMESPath 的介绍：JMESPath is a query language for JSON. JMESPath 有一套完备的语言规范，完善的测试用例以及各种编程语言的实现，用它可以构造出灵活强大的查询。官网上有一篇浅显易读的[教程文档](https://jmespath.org/tutorial.html)，并且配有示例和交互式环境来学习，半个小时就能上手，学到就是赚到！

## 应用

JMESPath 有什么用？

不论是前端还是后端，我们通过接口取到数据之后，往往都需要做数据的整理工作（按条件过滤，清理无用字段，构造新的结构，或是将层层嵌套的json扁平化等），使用 JMESPath 来完成这些工作真是再好不过了。特别是在命令行环境中，编写程序或脚本就相对困难一些，在这种情况下 JMESPath 也可以大显身手。

### 一个实际的应用场景

AWS CLI 是一个非常强大的命令行工具，可以实现几乎所有的 AWS 控制台操作。使用 AWS CLI 进行查询操作时，输出结果通常是 json 格式。这时，使用`--query`参数就可以传入 JMESPath 来对结果进行处理。AWS 的文档上有[一些示例](https://docs.aws.amazon.com/zh_cn/cli/latest/userguide/cli-usage-output.html#cli-usage-output-filter)。这里，我也举一个自己的例子。

目标：查询在指定 subnet 中的 RDS 实例。  
使用 AWS CLI 命令`aws rds describe-db-instances`可列出所有的 RDS 实例，结果输出样例如下：
```javascript
{
    'Marker': 'string',
    'DBInstances': [
        {
            'DBInstanceIdentifier': 'string',
            'DBInstanceClass': 'string',
            'Engine': 'string',
            'DBInstanceStatus': 'string',
            'MasterUsername': 'string',
            'DBName': 'string',
            'Endpoint': {
                'Address': 'string',
                'Port': 123,
                'HostedZoneId': 'string'
            },
            'AllocatedStorage': 123,
            'InstanceCreateTime': datetime(2015, 1, 1),
            'PreferredBackupWindow': 'string',
            'BackupRetentionPeriod': 123,
            'DBSecurityGroups': [
                {
                    'DBSecurityGroupName': 'string',
                    'Status': 'string'
                },
            ],
            'VpcSecurityGroups': [
                {
                    'VpcSecurityGroupId': 'string',
                    'Status': 'string'
                },
            ],
            'DBParameterGroups': [
                {
                    'DBParameterGroupName': 'string',
                    'ParameterApplyStatus': 'string'
                },
            ],
            'AvailabilityZone': 'string',
            'DBSubnetGroup': {
                'DBSubnetGroupName': 'string',
                'DBSubnetGroupDescription': 'string',
                'VpcId': 'string',
                'SubnetGroupStatus': 'string',
                'Subnets': [
                    {
                        'SubnetIdentifier': 'string',
                        'SubnetAvailabilityZone': {
                            'Name': 'string'
                        },
                        'SubnetStatus': 'string'
                    },
                ],
                'DBSubnetGroupArn': 'string'
            },
            'PreferredMaintenanceWindow': 'string',
            'PendingModifiedValues': {
                'DBInstanceClass': 'string',
                'AllocatedStorage': 123,
                'MasterUserPassword': 'string',
                'Port': 123,
                'BackupRetentionPeriod': 123,
                'MultiAZ': True|False,
                'EngineVersion': 'string',
                'LicenseModel': 'string',
                'Iops': 123,
                'DBInstanceIdentifier': 'string',
                'StorageType': 'string',
                'CACertificateIdentifier': 'string',
                'DBSubnetGroupName': 'string',
                'PendingCloudwatchLogsExports': {
                    'LogTypesToEnable': [
                        'string',
                    ],
                    'LogTypesToDisable': [
                        'string',
                    ]
                },
                'ProcessorFeatures': [
                    {
                        'Name': 'string',
                        'Value': 'string'
                    },
                ]
            },
            'LatestRestorableTime': datetime(2015, 1, 1),
            'MultiAZ': True|False,
            'EngineVersion': 'string',
            'AutoMinorVersionUpgrade': True|False,
            'ReadReplicaSourceDBInstanceIdentifier': 'string',
            'ReadReplicaDBInstanceIdentifiers': [
                'string',
            ],
            'ReadReplicaDBClusterIdentifiers': [
                'string',
            ],
            'LicenseModel': 'string',
            'Iops': 123,
            'OptionGroupMemberships': [
                {
                    'OptionGroupName': 'string',
                    'Status': 'string'
                },
            ],
            'CharacterSetName': 'string',
            'SecondaryAvailabilityZone': 'string',
            'PubliclyAccessible': True|False,
            'StatusInfos': [
                {
                    'StatusType': 'string',
                    'Normal': True|False,
                    'Status': 'string',
                    'Message': 'string'
                },
            ],
            'StorageType': 'string',
            'TdeCredentialArn': 'string',
            'DbInstancePort': 123,
            'DBClusterIdentifier': 'string',
            'StorageEncrypted': True|False,
            'KmsKeyId': 'string',
            'DbiResourceId': 'string',
            'CACertificateIdentifier': 'string',
            'DomainMemberships': [
                {
                    'Domain': 'string',
                    'Status': 'string',
                    'FQDN': 'string',
                    'IAMRoleName': 'string'
                },
            ],
            'CopyTagsToSnapshot': True|False,
            'MonitoringInterval': 123,
            'EnhancedMonitoringResourceArn': 'string',
            'MonitoringRoleArn': 'string',
            'PromotionTier': 123,
            'DBInstanceArn': 'string',
            'Timezone': 'string',
            'IAMDatabaseAuthenticationEnabled': True|False,
            'PerformanceInsightsEnabled': True|False,
            'PerformanceInsightsKMSKeyId': 'string',
            'PerformanceInsightsRetentionPeriod': 123,
            'EnabledCloudwatchLogsExports': [
                'string',
            ],
            'ProcessorFeatures': [
                {
                    'Name': 'string',
                    'Value': 'string'
                },
            ],
            'DeletionProtection': True|False,
            'AssociatedRoles': [
                {
                    'RoleArn': 'string',
                    'FeatureName': 'string',
                    'Status': 'string'
                },
            ],
            'ListenerEndpoint': {
                'Address': 'string',
                'Port': 123,
                'HostedZoneId': 'string'
            },
            'MaxAllocatedStorage': 123
        },
    ]
}
```

在上面的 AWS CLI 命令后加上`--query`参数即可实现目标效果，命令如下：
```
aws rds describe-db-instances --query 'DBInstances[].[DBInstanceIdentifier, DBSubnetGroup.Subnets[].SubnetIdentifier][*].{name: [0], nets: [1]}[?nets!=null && contains(nets, `subnet-12345678`)]'
```

解释：JMESPath 可以从左往右逐步计算结果，下面逐步解释。  
1. `DBInstances[]`  
    提取出结果中的`DBInstances`列表。  
2. `.[DBInstanceIdentifier, DBSubnetGroup.Subnets[].SubnetIdentifier]`
    MultiSelect语法，从`DBInstances`列表中去掉无用字段，仅保留`DBInstanceIdentifier`和`DBSubnetGroup结构中Subnets列表中的每一个SubnetIdentifier`字段。
3. `[*].{name: [0], nets: [1]}`
    按指定形式构造新的对象（结构）
4. `[?nets!=null && contains(nets, `subnet-12345678`)]`
    条件过滤，仅保留`nets`字段不为null且包含指定subnet-id的对象。

这个例子是一个较为复杂的综合场景，用到了 Projection，MultiSelect，Function等特性。从这里就可以看出 JMESPath 的灵活强大。

## 缺点

当然 JMESPath 也有它的缺点： 
1. 复杂逻辑编写困难（至少对于初学者来说是的） 
2. 可读性较差，想要一眼看出一个较复杂的 JMESPath 是什么意思并不容易。相比来说，Python 就更加友好一些。

还是上面的例子，换成 Python 来实现如下：  
先写一个 Lambda 函数实现上面 JMESPath 的功能：
```Python
f = lambda data, subnet_id: [instance['DBInstanceIdentifier'] for instance in data['DBInstances'] if instance.get('DBSubnetGroup') and instance['DBSubnetGroup']['Subnets'] and (subnet_id in [subnet['SubnetIdentifier'] for subnet in instance['DBSubnetGroup']['Subnets']])]
```

然后为了能在命令行中接收 AWS CLI 通过管道的输出，稍作改造，整个命令如下：
```Shell
aws rds describe-db-instances | python -c "import sys,json;content=sys.stdin.read();data=json.loads(content);f = lambda data, subnet_id: [instance['DBInstanceIdentifier'] for instance in data['DBInstances'] if instance.get('DBSubnetGroup') and instance['DBSubnetGroup']['Subnets'] and (subnet_id in [subnet['SubnetIdentifier'] for subnet in instance['DBSubnetGroup']['Subnets']])];r=f(data, 'subnet-12345678');print(r)"
```

以上可以达到相同的效果。

在这个例子中，为了写出正确的 JMESPath 我花了将近一个小时来调试，而写 Python 代码我用了不到一分钟。也许是我初学 JMESPath 还不够熟练的缘故吧。

## 总结

文章写到这里，我刚学习完 JMESPath 的那种兴奋也平复下来。好像用 Python 也能实现相同的效果嘛，而且语义似乎也更清晰一些。不过在上面的 Python Lambda 函数中，后置较长的 if 语句对阅读代码造成了一定的影响（需要前面看看后面看看），而 JMESPath 可以从左向右分段阅读，数据的处理流程类似于“管道”。大家怎么看呢？
