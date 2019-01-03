---
title: "AWS RDS Automatic Minor Version Upgrade(AMVU) 이슈 해결하기"
categories:
    - AWS
last_modified_at: 2019-01-03T18:26:00+09:00
# toc: true // toggle
author_profile: true
read_time: true
tags:
    - AWS
    - RDS
    - AMVU
--- 

회사에서 RDS 서비스를 애용하고 있던 중에, 작년 12월 23일경에 AWS팀으로부터 메일을 하나 받았습니다. 혹시나 운영하고 있는 리소스들이 문제가 생겼나 싶어서 급하게 확인을 했는데요. 지금 당장 생긴 문제는 아니였지만 확인을 안 했다면 꽤나 큰 문제가 있었을텐데, 다행히도 미리 발견했습니다.

## Automatic Minor Version Upgrade(AMNU)란?
말 그대로, 마이너 버전을 자동으로 업그레이드 시켜주는 일종의 AWS RDS의 기능입니다. 물론, 업그레이드를 할 때는 다운타임이 발생하게 됩니다.

## 문제의 시작
![사진1](https://blogfiles.pstatic.net/MjAxOTAxMDNfMTcw/MDAxNTQ2NTA3NjUxMTI3.yeDd3PTYGfPKnUoScxPHvAp-v27-IVarqx7tk_RNyhkg.jboyTbH46kOTjz7odNqkaAhK3sIthLZAG1B-Tl5_rmQg.PNG.broccoli98/1_1.PNG?type=w1)
> **Changes to Amazon RDS Automatic Minor Version Upgrade Behavior [AWS Account: 000000000]**
>> You are receiving this email because you have had one or more active Amazon RDS database instances with the AMVU property set to TRUE. <br/>
>> **If you do not want your database instance to be automatically upgraded to the default engine minor version, you can modify the database instance and set this property to FALSE.**

다음은 메일 제목과 내용의 일부입니다. 해석해보면, **하나 이상의 RDS 인스턴스의 AMVU속성이 True로 설정 되어 있다는 내용입니다. 그러니까 이에 따른 적절한 조치를 미리 취해달라는 내용인데요.** 큰 문제가 생겼습니다. 왜냐하면 지금 제가 회사에서 서비스 중인 모든 RDS 인스턴스는 무중단 서비스이기 때문에 다운타임이 전혀 없어야 하기 때문입니다.


## 해결
일단, 가장 빠르게 확인하기 위해서 AWS-CLI로 RDS 인스턴스 정보를 확인해봤습니다.
```
aws rds describe-db-instances \
--db-instance-identifier [RDS 인스턴스명]
```

### Output
```json
{
    "DBInstances": [
        {
            "PubliclyAccessible": true,
            "MasterUsername": "root",
            "MonitoringInterval": 0,
            "LicenseModel": "postgresql-license",
            "VpcSecurityGroups": [
                {
                    "Status": "active",
                    "VpcSecurityGroupId": "sg-000000000000000"
                },
            ],
            "InstanceCreateTime": "2018-12-06T09:49:34.199Z",
            "CopyTagsToSnapshot": true,
            "OptionGroupMemberships": [
                {
                    "Status": "in-sync",
                    "OptionGroupName": "default:postgres-10"
                }
            ],
            "PendingModifiedValues": {},
            "Engine": "postgres",
            "MultiAZ": false,
            "LatestRestorableTime": "2019-01-03T08:08:57Z",
            "DBSecurityGroups": [],
            "DBParameterGroups": [
                {
                    "DBParameterGroupName": "postgres10",
                    "ParameterApplyStatus": "in-sync"
                }
            ],
            "PerformanceInsightsEnabled": false,
            "AutoMinorVersionUpgrade": true,
            // ... 생략
            "DBInstanceClass": "db.t2.micro",
            "DbInstancePort": 0,
            "DBInstanceIdentifier": "RDS_NAME"
        }
    ]
}
```

> "AutoMinorVersionUpgrade": true,

RDS 인스턴스의 정보 중에 AMVU 설정이 아니나 다를까, true로 설정되어 있었습니다. 다시 명령어를 통해 설정을 변경해줬습니다.

```
aws rds modify-db-instance \
--db-instance-identifier [RDS 인스턴스명] \
--no-auto-minor-version-upgrade
```

### Output
```json
{
    "DBInstances": [
        {
            "PubliclyAccessible": true,
            "MasterUsername": "root",
            "MonitoringInterval": 0,
            "LicenseModel": "postgresql-license",
            "VpcSecurityGroups": [
                {
                    "Status": "active",
                    "VpcSecurityGroupId": "sg-000000000000000"
                },
            ],
            "InstanceCreateTime": "2018-12-06T09:49:34.199Z",
            "CopyTagsToSnapshot": true,
            "OptionGroupMemberships": [
                {
                    "Status": "in-sync",
                    "OptionGroupName": "default:postgres-10"
                }
            ],
            "PendingModifiedValues": {},
            "Engine": "postgres",
            "MultiAZ": false,
            "LatestRestorableTime": "2019-01-03T08:08:57Z",
            "DBSecurityGroups": [],
            "DBParameterGroups": [
                {
                    "DBParameterGroupName": "postgres10",
                    "ParameterApplyStatus": "in-sync"
                }
            ],
            "PerformanceInsightsEnabled": false,
            "AutoMinorVersionUpgrade": false,
            // ... 생략
            "DBInstanceClass": "db.t2.micro",
            "DbInstancePort": 0,
            "DBInstanceIdentifier": "RDS_NAME"
        }
    ]
}
```
> "AutoMinorVersionUpgrade": false,

## 마무리
Slack의 AWS 커뮤니티에서 활동 중에 이와 같은 내용으로 질문을 했던 사람이 있었는데 질문에 대한 답변 중에 이런 내용이 있었습니다.
> RDS를 원하는 시기에 원하는 방식으로 업그레이드하기 위하여 Auto minor upgrade 기능을 Off해서 사용하는 케이스가 흔한데, 점진적으로 이런 아키텍처/문화에서 탈피할 수 있으면 좋겠네요. - hanjin

이 말이 참 와닿았습니다. 그리고, 무조건적으로 마이너 버전을 업그레이드를 하지 않는게 아니고 적당한 시기에 적당한 수위에 업그레이드를 할 수 있도록 생각을 해봐야할 것 같습니다.

## 꿀팁
RDS 인스턴스가 굉장히 많다면 변경 사항을 하나하나씩 확인하기에도, 그렇다고 한꺼번에 확인하기에도 쉽지 않으실텐데요. RDS 인스턴스를 조회하는 명령어에 query 옵션만 넣는다면, 쉽게 내가 원하는 데이터만 뽑아낼 수 있습니다.

```
aws rds describe-db-instances \
--query 'DBInstances[*].{Name:DBInstanceIdentifier, AMVU:AutoMinorVersionUpgrade}'
```
query 옵션의 내용은 다음과 같습니다.
> 모든 DB 인스턴스를 조회하는데, DBInstanceIdentifier 항목의 데이터는 Name이라는 항목으로, AutoMinorVersionUpgrade 항목의 데이터는 AMVU이라는 항목으로 보여줘

### Output
```json
[
    {
        "Name": "RDS_instance_1",
        "AMVU": true
    },
    {
        "Name": "RDS_instance_2",
        "AMVU": false
    },
    {
        "Name": "RDS_instance_3",
        "AMVU": false
    }
]
```