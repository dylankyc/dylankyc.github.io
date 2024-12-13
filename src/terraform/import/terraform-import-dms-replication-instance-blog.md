# Manage AWS DMS resource using terraform

我们如何管理云服务中现有的资源呢？此时需要 `terraform import` 来帮忙。

我们拿 AWS DMS（Data Migration Service）来举例。

一般来讲，DMS 会有复制实例（Replication Instance）用来执行数据的迁移，下面我们来通过 terraform 来管理一个现有的复制实例。

从 AWS console 中查询需要管理的复制实例的 ARN：`arn:aws-cn:dms:cn-northwest-1:501502503504:rep:DOYOUTHINKTERRAFORMISAGOODTOOL`，因为 `import` 需要用到这个资源 ID。

下面开始 import。

首先，给 `terraform` 来个 `alias`:

```bash
alias t=terraform
```

其次，运行 terraform import:

```bash
t import aws_dms_replication_instance.feature arn:aws-cn:dms:cn-northwest-1:501502503504:rep:DOYOUTHINKTERRAFORMISAGOODTOOL
```

输出:

```
aws_dms_replication_instance.feature: Importing from ID "arn:aws-cn:dms:cn-northwest-1:501502503504:rep:DOYOUTHINKTERRAFORMISAGOODTOOL"...
aws_dms_replication_instance.feature: Import prepared!
Prepared aws_dms_replication_instance for import
aws_dms_replication_instance.feature: Refreshing state... [id=arn:aws-cn:dms:cn-northwest-1:501502503504:rep:DOYOUTHINKTERRAFORMISAGOODTOOL]
╷
│ Error: error describing DMS Replication Instance (arn:aws-cn:dms:cn-northwest-1:501502503504:rep:DOYOUTHINKTERRAFORMISAGOODTOOL): InvalidParameterValueException: The parameter value arn:aws-cn:dms:cn-northwest-1:501502503504:rep:DOYOUTHINKTERRAFORMISAGOODTOOL is not valid for argument Filter: replication-instance-id due to its length 86 exceeds 63.
│ status code: 400, request id: a6aaaaa3-1aaf-4aa6-aaa3-1aaaaaaaaaa5
│
│

Not set region correctly.
```

什么？竟然出错了: `replication-instance-id due to its length 86 exceeds 63`：

```
│ Error: error describing DMS Replication Instance (arn:aws-cn:dms:cn-northwest-1:501502503504:rep:DOYOUTHINKTERRAFORMISAGOODTOOL): InvalidParameterValueException: The parameter value arn:aws-cn:dms:cn-northwest-1:501502503504:rep:DOYOUTHINKTERRAFORMISAGOODTOOL is not valid for argument Filter: replication-instance-id due to its length 86 exceeds 63.
```

这个长度超过 63 的错是什么意思？

经过 google 半小时的查询，终于恍然大悟，原来 import 的参数不是 ARN，而是一个 ID。那这个 ID 在 AWS 里就是资源的 identifier。

于是修改 import 参数，传递 DMS 复制实例的 identifier，再次运行 `terraform import`。

```
t import aws_dms_replication_instance.feature featrue-dms-test
```

发现还是出错了:

```
aws_dms_replication_instance.feature: Importing from ID "featrue-dms-test"...
aws_dms_replication_instance.feature: Import prepared!
  Prepared aws_dms_replication_instance for import
aws_dms_replication_instance.feature: Refreshing state... [id=featrue-dms-test]
╷
│ Error: Cannot import non-existent remote object
│
│ While attempting to import an existing object to "aws_dms_replication_instance.feature", the provider detected that no object
│ exists with the given id. Only pre-existing objects can be imported; check that the id is correct and that it is associated with
│ the provider's configured region or endpoint, or use "terraform apply" to create a new remote object for this resource.
    ╵
```

这次不是报长度的错了，而是 `the provider detected that no object exists with the given id`。

这个错误又是什么意思呢？

再经历了又半小时的 google 之后，发现可能是没有设置正确的 region 导致的，于是讲 AWS_DEFAULT_REGION 设置成正确的值之后，运行 import:

```
set_up_aws_profile_sandbox_and_region
t import aws_dms_replication_instance.feature featrue-dms-test
```

输出：

```
aws_dms_replication_instance.feature: Importing from ID "featrue-dms-test"...
aws_dms_replication_instance.feature: Import prepared!
  Prepared aws_dms_replication_instance for import
aws_dms_replication_instance.feature: Refreshing state... [id=featrue-dms-test]

Import successful!

The resources that were imported are shown above. These resources are now in
your Terraform state and will henceforth be managed by Terraform.
```

Congratulations! 终于成功了！

下面看看我们有什么:

    ls -lh
    .rw-r--r--   57 William Shakespear 27 Jun 13:47 main.tf
    .rw-r--r-- 2.1k William Shakespear 30 Jun 10:27 terraform.tfstate

我们有一个 `main.tf` 的配置文件和 `terraform.tfstate` 的状态文件。于是我们可以运行 `terraform plan` 和 `terraform apply` 来创建或修改基础设施的变更了。值得注意的是，在执行 `terraform apply` 之前，会创建一个 `plan` ，这个 `plan` 是本地状态和云服务之间的差异，而 `apply` 则会将本地配置同步到云服务（此处为 AWS）。

下面我们运行 `terraform plan`。

就像 `terraform plan --help` 命令所说的:

> Generates a speculative execution plan, showing what actions Terraform
> would take to apply the current configuration. This command will not
> actually perform the planned actions.

> You can optionally save the plan to a file, which you can then pass to
> the "apply" command to perform exactly the actions described in the plan.

我们运行 `terraform plan`:

```
╷
│ Error: Missing required argument
│
│   on main.tf line 2, in resource "aws_dms_replication_instance" "feature":
│    2: resource "aws_dms_replication_instance" "feature" {
│
│ The argument "replication_instance_id" is required, but no definition was found.
╵
╷
│ Error: Missing required argument
│
│   on main.tf line 2, in resource "aws_dms_replication_instance" "feature":
│    2: resource "aws_dms_replication_instance" "feature" {
│
│ The argument "replication_instance_class" is required, but no definition was found.
```

出错啦！执行 `terraform plan` 提示了两个错: `replication_instance_class` 和 `replication_instance_id` 是必须的，我们没有提供。当然啦，我们的 `main.tf` 还是空空的状态。

> There's error when executing `terraform plan`, it shows `"replication_instance_id" is required, but no definition was found`. What is `replication_instance_id` anyway?

```hcl
resource "aws_dms_replication_instance" "feature" {
}
```

在调查了 terraform 文档之后，我们补齐这两个必须参数。

```hcl
resource "aws_dms_replication_instance" "feature" {
  replication_instance_id    = "feature-dms-test"
  replication_instance_class = "dms.r5.xlarge"
}
```

执行 `terraform plan`:

```
aws_dms_replication_instance.feature: Refreshing state... [id=featrue-dms-test]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following
symbols:
-/+ destroy and then create replacement

Terraform will perform the following actions:

  # aws_dms_replication_instance.feature must be replaced
-/+ resource "aws_dms_replication_instance" "feature" {
      ~ allocated_storage                = 500 -> (known after apply)
      ~ auto_minor_version_upgrade       = true -> (known after apply)
      ~ availability_zone                = "cn-northwest-1c" -> (known after apply)
      ~ engine_version                   = "3.4.6" -> (known after apply)
      ~ id                               = "featrue-dms-test" -> (known after apply)
      ~ kms_key_arn                      = "arn:aws-cn:kms:cn-northwest-1:501502503504:key/39999999-9999-9999-9999-999999999999" -> (known after apply)
      ~ multi_az                         = false -> (known after apply)
      ~ preferred_maintenance_window     = "mon:11:50-mon:12:20" -> (known after apply)
      ~ publicly_accessible              = false -> (known after apply)
      ~ replication_instance_arn         = "arn:aws-cn:dms:cn-northwest-1:501502503504:rep:ILOVEREADALLPOEMSOFWILLIAMSHAKESPEAR" -> (known after apply)
      ~ replication_instance_id          = "featrue-dms-test" -> "feature-dms-test" # forces replacement
      ~ replication_instance_private_ips = [
          - "100.200.100.1",
        ] -> (known after apply)
      ~ replication_instance_public_ips  = [
          - "",
        ] -> (known after apply)
      ~ replication_subnet_group_id      = "default-vpc-010203040506070809" -> (known after apply)
      - tags                             = {
          - "description" = "featrue-dms-test"
        } -> null
      ~ tags_all                         = {
          - "description" = "featrue-dms-test"
        } -> (known after apply)
      ~ vpc_security_group_ids           = [
          - "sg-a0b0c0d0a0b0c0d0",
        ] -> (known after apply)
        # (1 unchanged attribute hidden)

      - timeouts {}
    }

Plan: 1 to add, 0 to change, 1 to destroy.

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run
"terraform apply" now.

```

太好了！终于像模像样的输出了！

可是，等等！为什么有 `1 to add` 和 `1 to destroy` 呢？难不成要摧毁我云服务的资源嘛？太可怕了。我先冷静冷静，下一步到底改干啥，可不要发生 `rm -rf /` 的惨剧。

在喝完咖啡，思索了一阵之后，得知一个重要结论。

我本地的配置 `main.tf` 和云服务的配置不匹配，但是本地只配置了 `replication_instance_id` 和 `replication_instance_class`，如果我执行 `terraform apply`，我就告诉 `terraform`：请给我创建一个 dms replication instance 资源，它的 `replication_instance_id` 和 `replication_instance_class` 分别如配置所说，其他参数看着给。于是 `terraform` 比较来比较去，发现只能先云服务现有的给删了（`1 to destroy`）再给我们创建一个（`1 to add`）。

这当然不是我们需要的。

那么我们应该怎么做呢？当然是拷贝 `terraform plan` 的输出（主要是波浪线的部分，这部分是 change）到 `main.tf`，完成手动云服务到本地配置的反向同步。

```hcl
resource "aws_dms_replication_instance" "feature" {
  replication_instance_id    = "feature-dms-test"
  replication_instance_class = "dms.r5.xlarge"

      ~ allocated_storage                = 500 -> (known after apply)
      ~ auto_minor_version_upgrade       = true -> (known after apply)
      ~ availability_zone                = "cn-northwest-1c" -> (known after apply)
      ~ engine_version                   = "3.4.6" -> (known after apply)
      ~ id                               = "featrue-dms-test" -> (known after apply)
      ~ kms_key_arn                      = "arn:aws-cn:kms:cn-northwest-1:501502503504:key/39999999-9999-9999-9999-999999999999" -> (known after apply)
      ~ multi_az                         = false -> (known after apply)
      ~ preferred_maintenance_window     = "mon:11:50-mon:12:20" -> (known after apply)
      ~ publicly_accessible              = false -> (known after apply)
      ~ replication_instance_arn         = "arn:aws-cn:dms:cn-northwest-1:501502503504:rep:ILOVEREADALLPOEMSOFWILLIAMSHAKESPEAR" -> (known after apply)
      ~ replication_instance_id          = "featrue-dms-test" -> "feature-dms-test" # forces replacement
      ~ replication_instance_private_ips = [
          - "100.200.100.1",
        ] -> (known after apply)
      ~ replication_instance_public_ips  = [
          - "",
        ] -> (known after apply)
      ~ replication_subnet_group_id      = "default-vpc-010203040506070809" -> (known after apply)
      - tags                             = {
          - "description" = "featrue-dms-test"
        } -> null
      ~ tags_all                         = {
          - "description" = "featrue-dms-test"
        } -> (known after apply)
      ~ vpc_security_group_ids           = [
          - "sg-a0b0c0d0a0b0c0d0",
        ] -> (known after apply)
        # (1 unchanged attribute hidden)

      - timeouts {}
}
```

于是，我们根据 `terraform plan` 的输出，填充到 `main.tf` 文件里:

```
resource "aws_dms_replication_instance" "feature" {
  replication_instance_class = "dms.r5.xlarge"

      allocated_storage                = 500
      auto_minor_version_upgrade       = true
      availability_zone                = "cn-northwest-1c"
      engine_version                   = "3.4.6"
      id                               = "featrue-dms-test"
      kms_key_arn                      = "arn:aws-cn:kms:cn-northwest-1:501502503504:key/39999999-9999-9999-9999-999999999999"
      multi_az                         = false
      preferred_maintenance_window     = "mon:11:50-mon:12:20"
      publicly_accessible              = false
      replication_instance_arn         = "arn:aws-cn:dms:cn-northwest-1:501502503504:rep:ILOVEREADALLPOEMSOFWILLIAMSHAKESPEAR"
      replication_instance_id          = "feature-dms-test" # forces replacement
      replication_instance_private_ips = [
          "100.200.100.1",
        ]
      replication_instance_public_ips  = [
          "",
        ]
      replication_subnet_group_id      = "default-vpc-010203040506070809"
      tags                             =  null
      tags_all                         = {
          "description" = "featrue-dms-test"
      }
      vpc_security_group_ids           = [
          "sg-a0b0c0d0a0b0c0d0",
      ]

      timeouts {}
}

```

此时再次运行 `terraform plan`

```
╷
│ Error: Invalid or unknown key
│
│   with aws_dms_replication_instance.feature,
│   on main.tf line 9, in resource "aws_dms_replication_instance" "feature":
│    9:   id                           = "featrue-dms-test"
│
╵
╷
│ Error: Value for unconfigurable attribute
│
│   with aws_dms_replication_instance.feature,
│   on main.tf line 14, in resource "aws_dms_replication_instance" "feature":
│   14:   replication_instance_arn     = "arn:aws-cn:dms:cn-northwest-1:501502503504:rep:ILOVEREADALLPOEMSOFWILLIAMSHAKESPEAR"
│
│ Can't configure a value for "replication_instance_arn": its value will be decided automatically based on the result of applying
│ this configuration.
╵
╷
│ Error: Value for unconfigurable attribute
│
│   with aws_dms_replication_instance.feature,
│   on main.tf line 16, in resource "aws_dms_replication_instance" "feature":
│   16:   replication_instance_private_ips = [
│   17:     "100.200.100.1",
│   18:   ]
│
│ Can't configure a value for "replication_instance_private_ips": its value will be decided automatically based on the result of
│ applying this configuration.
╵
╷
│ Error: Value for unconfigurable attribute
│
│   with aws_dms_replication_instance.feature,
│   on main.tf line 19, in resource "aws_dms_replication_instance" "feature":
│   19:   replication_instance_public_ips = [
│   20:     "",
│   21:   ]
│
│ Can't configure a value for "replication_instance_public_ips": its value will be decided automatically based on the result of
│ applying this configuration.
╵
```

它会提示一些错，意思是有些参数会在创建时决定，所以我们无须提供，注释掉对应的行如 `replication_instance_private_ips` 和 `replication_instance_public_ips`，再次运行 `terraform plan`。

> Fix error by comment error line. As `replication_instance_private_ips` and `replication_instance_public_ips` will be decided automatically based on the result of applying this configuration. We comment it out and run plan again.

```
resource "aws_dms_replication_instance" "feature" {
  publicly_accessible          = false
  # replication_instance_arn     = "arn:aws-cn:dms:cn-northwest-1:501502503504:rep:ILOVEREADALLPOEMSOFWILLIAMSHAKESPEAR"
  replication_instance_id = "feature-dms-test" # forces replacement
  # replication_instance_private_ips = [
  #   "100.200.100.1",
  # ]
  # replication_instance_public_ips = [
  #   "",
  # ]
}
```

但是我们还是会发现出错了，这是为什么呢？

> But we also encounter an error, saying we will destroy the resource and recreate it.

```
aws_dms_replication_instance.feature: Refreshing state... [id=featrue-dms-test]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following
symbols:
-/+ destroy and then create replacement

Terraform will perform the following actions:

  # aws_dms_replication_instance.feature must be replaced
-/+ resource "aws_dms_replication_instance" "feature" {
      ~ id                               = "featrue-dms-test" -> (known after apply)
      ~ replication_instance_arn         = "arn:aws-cn:dms:cn-northwest-1:501502503504:rep:ILOVEREADALLPOEMSOFWILLIAMSHAKESPEAR" -> (known after apply)
      ~ replication_instance_id          = "featrue-dms-test" -> "feature-dms-test" # forces replacement
      ~ replication_instance_private_ips = [
          - "100.200.100.1",
        ] -> (known after apply)
      ~ replication_instance_public_ips  = [
          - "",
        ] -> (known after apply)
      - tags                             = {
          - "description" = "featrue-dms-test"
        } -> null
      ~ tags_all                         = {
          - "description" = "featrue-dms-test"
        } -> (known after apply)
        # (11 unchanged attributes hidden)

        # (1 unchanged block hidden)
    }

Plan: 1 to add, 0 to change, 1 to destroy.

```

我们尝试这删除本地 `terraform.tfstate` 文件，运行 `terraform plan`，发现错误依旧。

> Delete terraform.state file and run `t import aws_dms_replication_instance.feature feature-dms-test` with correct name.

```
aws_dms_replication_instance.feature: Importing from ID "feature-dms-test"...
aws_dms_replication_instance.feature: Import prepared!
  Prepared aws_dms_replication_instance for import
aws_dms_replication_instance.feature: Refreshing state... [id=feature-dms-test]
╷
│ Error: Cannot import non-existent remote object
│
│ While attempting to import an existing object to "aws_dms_replication_instance.feature", the provider detected that no object
│ exists with the given id. Only pre-existing objects can be imported; check that the id is correct and that it is associated with
│ the provider's configured region or endpoint, or use "terraform apply" to create a new remote object for this resource.

```

啊！细心的我终于发现是因为云服务的资源名字被叫错了！云服务的 identifier 叫做 `featrue-dms-test`，而我们需要 `import` 的名字叫做 `feature-dms-test`！我的心里唱起了深深太平洋底深深伤心，于是紧急联系相关人员这个名字是否可以更改，在得知这只是个测试任务可以改名字时，我果断（含泪）敲下了如下的命令:

> Change dms replication instance identifier.

```bash
aws dms modify-replication-instance --replication-instance-arn arn:aws-cn:dms:cn-northwest-1:501502503504:rep:ILOVEREADALLPOEMSOFWILLIAMSHAKESPEAR --replication-instance-identifier feature-dms-test --apply-immediately
```

这是它的输出:

```
{
    "ReplicationInstance": {
        "ReplicationInstanceIdentifier": "feature-dms-test",
        "ReplicationInstanceClass": "dms.r5.xlarge",
        "ReplicationInstanceStatus": "available",
        "AllocatedStorage": 500,
        "InstanceCreateTime": "2022-05-23T12:59:24.006000+08:00",
        "VpcSecurityGroups": [
            {
                "VpcSecurityGroupId": "sg-a0b0c0d0a0b0c0d0",
                "Status": "active"
            }
        ],
        "AvailabilityZone": "cn-northwest-1c",
        "ReplicationSubnetGroup": {
            "ReplicationSubnetGroupIdentifier": "default-vpc-010203040506070809",
            "ReplicationSubnetGroupDescription": "default group created by console for vpc id vpc-010203040506070809",
            "VpcId": "vpc-010203040506070809",
            "SubnetGroupStatus": "Complete",
            "Subnets": [
                {
                    "SubnetIdentifier": "subnet-0102030405060708",
                    "SubnetAvailabilityZone": {
                        "Name": "cn-northwest-1b"
                    },
                    "SubnetStatus": "Active"
                },
                {
                    "SubnetIdentifier": "subnet-0807060504030201",
                    "SubnetAvailabilityZone": {
                        "Name": "cn-northwest-1c"
                    },
                    "SubnetStatus": "Active"
                },
                {
                    "SubnetIdentifier": "subnet-1213141516171819",
                    "SubnetAvailabilityZone": {
                        "Name": "cn-northwest-1a"
                    },
                    "SubnetStatus": "Active"
                }
            ]
        },
        "PreferredMaintenanceWindow": "mon:11:50-mon:12:20",
        "PendingModifiedValues": {},
        "MultiAZ": false,
        "EngineVersion": "3.4.6",
        "AutoMinorVersionUpgrade": true,
        "KmsKeyId": "arn:aws-cn:kms:cn-northwest-1:501502503504:key/99999999999c6-4999999999999999999999",
        "ReplicationInstanceArn": "arn:aws-cn:dms:cn-northwest-1:501502503504:rep:ILOVEREADALLPOEMSOFWILLIAMSHAKESPEAR",
        "ReplicationInstancePrivateIpAddress": "100.200.100.1",
        "ReplicationInstancePublicIpAddresses": [
            null
        ],
        "ReplicationInstancePrivateIpAddresses": [
            "100.200.100.1"
        ],
        "PubliclyAccessible": false
    }
}
```

等待了片刻之后，identifier 被修改成功了。赶紧试试 `terraform import` 吧～

```
t import aws_dms_replication_instance.feature feature-dms-test
```

终于 `import` 成功了:

```
aws_dms_replication_instance.feature: Importing from ID "feature-dms-test"...
aws_dms_replication_instance.feature: Import prepared!
  Prepared aws_dms_replication_instance for import
aws_dms_replication_instance.feature: Refreshing state... [id=feature-dms-test]

Import successful!

The resources that were imported are shown above. These resources are now in
your Terraform state and will henceforth be managed by Terraform.


```

此时的 `main.tf` 依旧空空如也，我们仿照先前的做法，将 `terraform plan` 的输出写到 `main.tf` 里。

```
t plan
```

输出:

```
aws_dms_replication_instance.feature: Refreshing state... [id=feature-dms-test]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following
symbols:
  ~ update in-place

Terraform will perform the following actions:

  # aws_dms_replication_instance.feature will be updated in-place
  ~ resource "aws_dms_replication_instance" "feature" {
        id                               = "feature-dms-test"
      ~ tags                             = {
          - "description" = "featrue-dms-test" -> null
        }
      ~ tags_all                         = {
          - "description" = "featrue-dms-test"
        } -> (known after apply)
        # (15 unchanged attributes hidden)

        # (1 unchanged block hidden)
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

我们看到，`add` 和 `destroy` 都变成 0 了，一个很大的进步！这就是说，我们即使执行 `terraform apply` 也不会误操作发生删除资源的事故。

下面要做的就是修复那些 `modify` 的部分（波浪线），如果无关紧要的话（比如 tag 之类的）可以放着不管，如果有洁癖的话（遵循 best practice），可以在 `main.tf` 里做些调整。

> We can see both add and destroy number is 0, we don't need to worry about any resource will be deleted, which may cause desaster, and it's best practice to add description to aws resource.

> Now we know there's no add or destroy, we can apply the changes by runing `terraform apply`. Notice, you need to enter `yes` to perform the actions.

此时运行 `t apply`:

输出:

```
aws_dms_replication_instance.feature: Refreshing state... [id=feature-dms-test]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following
symbols:
  ~ update in-place

Terraform will perform the following actions:

  # aws_dms_replication_instance.feature will be updated in-place
  ~ resource "aws_dms_replication_instance" "feature" {
        id                               = "feature-dms-test"
      ~ tags                             = {
          - "description" = "featrue-dms-test" -> null
        }
      ~ tags_all                         = {
          - "description" = "featrue-dms-test"
        } -> (known after apply)
        # (15 unchanged attributes hidden)

        # (1 unchanged block hidden)
    }

Plan: 0 to add, 1 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

aws_dms_replication_instance.feature: Modifying... [id=feature-dms-test]
aws_dms_replication_instance.feature: Modifications complete after 0s [id=feature-dms-test]

Apply complete! Resources: 0 added, 1 changed, 0 destroyed.
```

此时 apply 成功，我们把一个 AWS DMS 的复制示例通过 terraform 来管理了，是不是很简单（真的有吗）？

> After applying the changes, we can check if there's any difference between our local infrastructure and the infrastructure in the cloud by running `terraform plan` again.

在执行完 `terraform apply` 之后，我们再次执行 `terraform plan`，它会因为检测不到变更（本地配置已经和云服务同步）而不做其他操作。

```
aws_dms_replication_instance.feature: Refreshing state... [id=feature-dms-test]

No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration and found no differences, so no changes are needed.
```

这样，我们就可以如法炮制 `import` 其他的资源了。

> Life is hard? Life is hard.

terraform doc:

https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/dms_replication_instance
