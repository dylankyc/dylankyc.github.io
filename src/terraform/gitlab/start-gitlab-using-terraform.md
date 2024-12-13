# Start gitlab using terraform

- [aws](#aws)
  - [aws provider](#aws-provider)
  - [vpc](#vpc)
- [ec2 instance](#ec2-instance)
- [security group](#security-group)
- [ebs volume](#ebs-volume)
- [eip](#eip)
- [gitlab setup](#gitlab-setup)
- [GitLab clone](#gitlab-clone)
- [GitLab administration](#gitlab-administration)

如何在 `AWS` 中快速部署一台 `GitLab` 服务？利用 `terraform`，我们可以自由调配云服务的资源，并快速将 `GitLab` 部署到 `AWS` 中。

# aws

## aws provider

在我们使用 `terraform` 创建云服务器时，我们需要指定 provider。

首先，我们定义 aws provider。

aws provider 是 `terraform` 的一个特殊的资源类型，它提供了一个简单的 API，用于访问 `AWS` 中的服务。

以下配置是 aws provider 的配置：

```hcl
provider "aws" {
  region = "cn-northwest-1"
}
```

我们指定了 `AWS` 的 `region` 为 `cn-northwest-1`，这个 `region` 就是我们的云服务器所在的地区。

## vpc

运行 `GitLab` 的 EC2 放在哪里呢？一般来讲，服务器需要一个 `VPC`，这个 `VPC` 就是我们的云服务器所在的网络，而我们的 EC2 instance 就是在这个 `VPC` 中的一个虚拟机。

一般我们会利用现有的 vpc 而不是新建 vpc，从 AWS console 中查询以下 vpc 的 id，我们把它放到名为 vpc 的 variable 中。

EN: As we have already have a vpc, we can use it to deploy our application. Here we use variable to define the vpc id.

```hcl
variable "vpc" {
  type        = string
  default     = "vpc-0f0f0f0f0f0f0f0f"
  description = "The VPC ID of "
}
```

# ec2 instance

在确定了 `VPC` 的 id 之后，我们就可以创建 EC2 instance 了。

我们使用 `aws_instance` 资源来创建 EC2 instance，并且指定了一个名为 `gitlab` 的实例。

此外，我们还指定了一个实例的类型，这个类型是 `m5a.large`，这个类型是 `AWS` 中的一个预定义的类型，我们可以在 AWS console 中查询到。

`ami` 为版本为 Ubuntu 20.04 的镜像，`key_name` 为登录 EC2 的 key，`root_block_device` 为系统盘，我们此时分配了 `40G` 的磁盘空间，`subnet_id` 为 vpc 下的一个子网，它也是预先就创建好的。

`vpc_security_group_ids` 为我们创建 EC2 所需的安全组，下面我们会讲它开启了哪些规则。

EN: We can use the ec2 resource to deploy our application. We use `aws_instance` resource and specify the vpc id, subnet id, instance type(`m5a.large` ), key_name(`gitlab`), Ubuntu 20.04 server(`ami-ffff111db56e65f8d`), 40GiB root block size volume.

```hcl
resource "aws_instance" "gitlab" {
  # Ubuntu Server 20.04 LTS (HVM), SSD Volume Type - ami-ffff111db56e65f8d (64-bit x86) / ami-0429c857c8db3027a (64-bit Arm)
  ami           = "ami-ffff111db56e65f8d"
  instance_type = "m5a.large"
  key_name      = "gitlab"

  root_block_device {
    volume_size = "40"
    volume_type = "gp3"
  }

  # (subnet-public1-cn-north-1a)
  subnet_id              = "subnet-2222333344445555"
  vpc_security_group_ids = ["${aws_security_group.gitlab.id}"]
  # associate_public_ip_address = true
  tags = {
    Name = "gitlab"
  }
}
```

在 EC2 创建好之后，我们可以这样登录:

EN: After provisioning, you can access the GitLab instance.

```
ssh -i ~/.ssh/gitlab.pem <public_ip>
```

# security group

EC2 实例需要一个安全组，用来控制它的 `ingress` 规则和 `egress` 规则。可以把 security group 看成是防火墙。

通过配置文件可知，我们允许 ssh 登录 EC2，允许访问 `GitLab` http 的端口 80 和 https 的端口 443。

对于出口的规则，我们不做任何限制，可以访问任何地方。

```hcl
resource "aws_security_group" "gitlab" {
  description = "Security group for gitlab"
  vpc_id      = var.vpc

  egress {
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound traffic"
    from_port   = 0
    protocol    = "-1"
    self        = false
    to_port     = 0
  }

  ingress {
    cidr_blocks = ["0.0.0.0/0"]
    description = "ssh"
    from_port   = 22
    protocol    = "tcp"
    self        = false
    to_port     = 22
  }

  ingress {
    cidr_blocks = ["0.0.0.0/0"]
    description = "allow public access http"
    from_port   = 80
    protocol    = "tcp"
    self        = false
    to_port     = 80
  }

  ingress {
    cidr_blocks = ["0.0.0.0/0"]
    description = "allow public access https"
    from_port   = 443
    protocol    = "tcp"
    self        = false
    to_port     = 443
  }

  name                   = "gitlab"
  revoke_rules_on_delete = false
  tags = {
    "Name" = "gitlab"
  }
  tags_all = {
    "Name" = "gitlab"
  }

  timeouts {}
}
```

# ebs volume

为了存储 git repository，我们可以利用 `aws_ebs_volume` 给 EC2 实例分配一个 EBS 磁盘，并且指定它的大小为 `40GiB`，再通过 `aws_volume_attachment` 将 EBS 磁盘与 EC2 实例进行绑定。

```hcl
resource "aws_ebs_volume" "gitlab" {
  availability_zone = "cn-north-1a"
  size              = 40
  type              = "gp3"

  tags = {
    Name = "gitlab"
  }
}

resource "aws_volume_attachment" "ebs_attachment_gitlab" {
  device_name = "/dev/sdh"
  volume_id   = aws_ebs_volume.gitlab.id
  instance_id = aws_instance.gitlab.id
}
```

好了，就这样就差不都了，我们来执行一下 `terraform plan` 命令，看看有没有错误。

下面是添加了 `aws_ebs_volume` 和 `aws_volume_attachment` 之后执行 `terraform plan` 的结果：

```
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_ebs_volume.gitlab will be created
  + resource "aws_ebs_volume" "gitlab" {
      + arn               = (known after apply)
      + availability_zone = "cn-north-1a"
      + encrypted         = (known after apply)
      + id                = (known after apply)
      + iops              = (known after apply)
      + kms_key_id        = (known after apply)
      + size              = 40
      + snapshot_id       = (known after apply)
      + tags              = {
          + "Name" = "gitlab"
        }
      + tags_all          = {
          + "Name" = "gitlab"
        }
      + throughput        = (known after apply)
      + type              = "gp3"
    }

  # aws_instance.gitlab will be created
  + resource "aws_instance" "gitlab" {
      + ami                                  = "ami-ffff111db56e65f8d"
      + arn                                  = (known after apply)
      + associate_public_ip_address          = (known after apply)
      + availability_zone                    = (known after apply)
      + cpu_core_count                       = (known after apply)
      + cpu_threads_per_core                 = (known after apply)
      + disable_api_termination              = (known after apply)
      + ebs_optimized                        = (known after apply)
      + get_password_data                    = false
      + host_id                              = (known after apply)
      + id                                   = (known after apply)
      + instance_initiated_shutdown_behavior = (known after apply)
      + instance_state                       = (known after apply)
      + instance_type                        = "m5a.large"
      + ipv6_address_count                   = (known after apply)
      + ipv6_addresses                       = (known after apply)
      + key_name                             = "gitlab"
      + monitoring                           = (known after apply)
      + outpost_arn                          = (known after apply)
      + password_data                        = (known after apply)
      + placement_group                      = (known after apply)
      + placement_partition_number           = (known after apply)
      + primary_network_interface_id         = (known after apply)
      + private_dns                          = (known after apply)
      + private_ip                           = (known after apply)
      + public_dns                           = (known after apply)
      + public_ip                            = (known after apply)
      + secondary_private_ips                = (known after apply)
      + security_groups                      = (known after apply)
      + source_dest_check                    = true
      + subnet_id                            = "subnet-88ff88ff88ff"
      + tags                                 = {
          + "Name" = "gitlab"
        }
      + tags_all                             = {
          + "Name" = "gitlab"
        }
      + tenancy                              = (known after apply)
      + user_data                            = (known after apply)
      + user_data_base64                     = (known after apply)
      + user_data_replace_on_change          = false
      + vpc_security_group_ids               = (known after apply)

      + capacity_reservation_specification {
          + capacity_reservation_preference = (known after apply)

          + capacity_reservation_target {
              + capacity_reservation_id = (known after apply)
            }
        }

      + ebs_block_device {
          + delete_on_termination = (known after apply)
          + device_name           = (known after apply)
          + encrypted             = (known after apply)
          + iops                  = (known after apply)
          + kms_key_id            = (known after apply)
          + snapshot_id           = (known after apply)
          + tags                  = (known after apply)
          + throughput            = (known after apply)
          + volume_id             = (known after apply)
          + volume_size           = (known after apply)
          + volume_type           = (known after apply)
        }

      + enclave_options {
          + enabled = (known after apply)
        }

      + ephemeral_block_device {
          + device_name  = (known after apply)
          + no_device    = (known after apply)
          + virtual_name = (known after apply)
        }

      + metadata_options {
          + http_endpoint               = (known after apply)
          + http_put_response_hop_limit = (known after apply)
          + http_tokens                 = (known after apply)
          + instance_metadata_tags      = (known after apply)
        }

      + network_interface {
          + delete_on_termination = (known after apply)
          + device_index          = (known after apply)
          + network_interface_id  = (known after apply)
        }

      + root_block_device {
          + delete_on_termination = true
          + device_name           = (known after apply)
          + encrypted             = (known after apply)
          + iops                  = (known after apply)
          + kms_key_id            = (known after apply)
          + throughput            = (known after apply)
          + volume_id             = (known after apply)
          + volume_size           = 60
          + volume_type           = "standard"
        }
    }

  # aws_security_group.gitlab will be created
  + resource "aws_security_group" "gitlab" {
      + arn                    = (known after apply)
      + description            = "Security group for gitlab"
      + egress                 = [
          + {
              + cidr_blocks      = [
                  + "0.0.0.0/0",
                ]
              + description      = "Allow all outbound traffic"
              + from_port        = 0
              + ipv6_cidr_blocks = []
              + prefix_list_ids  = []
              + protocol         = "-1"
              + security_groups  = []
              + self             = false
              + to_port          = 0
            },
        ]
      + id                     = (known after apply)
      + ingress                = [
          + {
              + cidr_blocks      = [
                  + "0.0.0.0/0",
                ]
              + description      = "allow public access http"
              + from_port        = 80
              + ipv6_cidr_blocks = []
              + prefix_list_ids  = []
              + protocol         = "tcp"
              + security_groups  = []
              + self             = false
              + to_port          = 80
            },
          + {
              + cidr_blocks      = [
                  + "0.0.0.0/0",
                ]
              + description      = "allow public access https"
              + from_port        = 443
              + ipv6_cidr_blocks = []
              + prefix_list_ids  = []
              + protocol         = "tcp"
              + security_groups  = []
              + self             = false
              + to_port          = 443
            },
          + {
              + cidr_blocks      = [
                  + "0.0.0.0/0",
                ]
              + description      = "test"
              + from_port        = 22
              + ipv6_cidr_blocks = []
              + prefix_list_ids  = []
              + protocol         = "tcp"
              + security_groups  = []
              + self             = false
              + to_port          = 22
            },
          + {
              + cidr_blocks      = [
                  + "10.0.0.0/16",
                ]
              + description      = "allow ssh"
              + from_port        = 22
              + ipv6_cidr_blocks = []
              + prefix_list_ids  = []
              + protocol         = "tcp"
              + security_groups  = []
              + self             = false
              + to_port          = 22
            },
        ]
      + name                   = "gitlab"
      + name_prefix            = (known after apply)
      + owner_id               = (known after apply)
      + revoke_rules_on_delete = false
      + tags                   = {
          + "Name" = "gitlab"
        }
      + tags_all               = {
          + "Name" = "gitlab"
        }
      + vpc_id                 = "vpc-0afafafafaf"

      + timeouts {}
    }

  # aws_volume_attachment.ebs_attachment_gitlab will be created
  + resource "aws_volume_attachment" "ebs_attachment_gitlab" {
      + device_name = "/dev/sdh"
      + id          = (known after apply)
      + instance_id = (known after apply)
      + volume_id   = (known after apply)
    }

Plan: 4 to add, 0 to change, 0 to destroy.

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.
```

以上显示，terraform 会创建以下 4 种资源:

- `aws_instance.gitlab`
- `aws_volume.gitlab`
- `aws_volume_attachment.ebs_attachment_gitlab`
- `aws_security_group.gitlab`

下面执行 `terraform apply` 命令，4 个资源都会被创建，并且资源的属性都会被设置为预期的值。

```
    Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
      + create

    Terraform will perform the following actions:

      # aws_ebs_volume.gitlab will be created
      + resource "aws_ebs_volume" "gitlab" {
          + arn               = (known after apply)
          + availability_zone = "cn-north-1a"
          + encrypted         = (known after apply)
          + id                = (known after apply)
          + iops              = (known after apply)
          + kms_key_id        = (known after apply)
          + size              = 40
          + snapshot_id       = (known after apply)
          + tags              = {
              + "Name" = "gitlab"
            }
          + tags_all          = {
              + "Name" = "gitlab"
            }
          + throughput        = (known after apply)
          + type              = "gp3"
        }

      # aws_instance.gitlab will be created
      + resource "aws_instance" "gitlab" {
          + ami                                  = "ami-ffff111db56e65f8d"
          + arn                                  = (known after apply)
          + associate_public_ip_address          = (known after apply)
          + availability_zone                    = (known after apply)
          + cpu_core_count                       = (known after apply)
          + cpu_threads_per_core                 = (known after apply)
          + disable_api_termination              = (known after apply)
          + ebs_optimized                        = (known after apply)
          + get_password_data                    = false
          + host_id                              = (known after apply)
          + id                                   = (known after apply)
          + instance_initiated_shutdown_behavior = (known after apply)
          + instance_state                       = (known after apply)
          + instance_type                        = "m5a.large"
          + ipv6_address_count                   = (known after apply)
          + ipv6_addresses                       = (known after apply)
          + key_name                             = "gitlab"
          + monitoring                           = (known after apply)
          + outpost_arn                          = (known after apply)
          + password_data                        = (known after apply)
          + placement_group                      = (known after apply)
          + placement_partition_number           = (known after apply)
          + primary_network_interface_id         = (known after apply)
          + private_dns                          = (known after apply)
          + private_ip                           = (known after apply)
          + public_dns                           = (known after apply)
          + public_ip                            = (known after apply)
          + secondary_private_ips                = (known after apply)
          + security_groups                      = (known after apply)
          + source_dest_check                    = true
          + subnet_id                            = "subnet-069a0f9b9c9f9f9f9"
          + tags                                 = {
              + "Name" = "gitlab"
            }
          + tags_all                             = {
              + "Name" = "gitlab"
            }
          + tenancy                              = (known after apply)
          + user_data                            = (known after apply)
          + user_data_base64                     = (known after apply)
          + user_data_replace_on_change          = false
          + vpc_security_group_ids               = (known after apply)

          + capacity_reservation_specification {
              + capacity_reservation_preference = (known after apply)

              + capacity_reservation_target {
                  + capacity_reservation_id = (known after apply)
                }
            }

          + ebs_block_device {
              + delete_on_termination = (known after apply)
              + device_name           = (known after apply)
              + encrypted             = (known after apply)
              + iops                  = (known after apply)
              + kms_key_id            = (known after apply)
              + snapshot_id           = (known after apply)
              + tags                  = (known after apply)
              + throughput            = (known after apply)
              + volume_id             = (known after apply)
              + volume_size           = (known after apply)
              + volume_type           = (known after apply)
            }

          + enclave_options {
              + enabled = (known after apply)
            }

          + ephemeral_block_device {
              + device_name  = (known after apply)
              + no_device    = (known after apply)
              + virtual_name = (known after apply)
            }

          + metadata_options {
              + http_endpoint               = (known after apply)
              + http_put_response_hop_limit = (known after apply)
              + http_tokens                 = (known after apply)
              + instance_metadata_tags      = (known after apply)
            }

          + network_interface {
              + delete_on_termination = (known after apply)
              + device_index          = (known after apply)
              + network_interface_id  = (known after apply)
            }

          + root_block_device {
              + delete_on_termination = true
              + device_name           = (known after apply)
              + encrypted             = (known after apply)
              + iops                  = (known after apply)
              + kms_key_id            = (known after apply)
              + throughput            = (known after apply)
              + volume_id             = (known after apply)
              + volume_size           = 60
              + volume_type           = "standard"
            }
        }

      # aws_security_group.gitlab will be created
      + resource "aws_security_group" "gitlab" {
          + arn                    = (known after apply)
          + description            = "Security group for gitlab"
          + egress                 = [
              + {
                  + cidr_blocks      = [
                      + "0.0.0.0/0",
                    ]
                  + description      = "Allow all outbound traffic"
                  + from_port        = 0
                  + ipv6_cidr_blocks = []
                  + prefix_list_ids  = []
                  + protocol         = "-1"
                  + security_groups  = []
                  + self             = false
                  + to_port          = 0
                },
            ]
          + id                     = (known after apply)
          + ingress                = [
              + {
                  + cidr_blocks      = [
                      + "0.0.0.0/0",
                    ]
                  + description      = "allow public access http"
                  + from_port        = 80
                  + ipv6_cidr_blocks = []
                  + prefix_list_ids  = []
                  + protocol         = "tcp"
                  + security_groups  = []
                  + self             = false
                  + to_port          = 80
                },
              + {
                  + cidr_blocks      = [
                      + "0.0.0.0/0",
                    ]
                  + description      = "allow public access https"
                  + from_port        = 443
                  + ipv6_cidr_blocks = []
                  + prefix_list_ids  = []
                  + protocol         = "tcp"
                  + security_groups  = []
                  + self             = false
                  + to_port          = 443
                },
              + {
                  + cidr_blocks      = [
                      + "0.0.0.0/0",
                    ]
                  + description      = "test"
                  + from_port        = 22
                  + ipv6_cidr_blocks = []
                  + prefix_list_ids  = []
                  + protocol         = "tcp"
                  + security_groups  = []
                  + self             = false
                  + to_port          = 22
                },
              + {
                  + cidr_blocks      = [
                      + "10.0.0.0/16",
                    ]
                  + description      = "allow ssh"
                  + from_port        = 22
                  + ipv6_cidr_blocks = []
                  + prefix_list_ids  = []
                  + protocol         = "tcp"
                  + security_groups  = []
                  + self             = false
                  + to_port          = 22
                },
            ]
          + name                   = "gitlab"
          + name_prefix            = (known after apply)
          + owner_id               = (known after apply)
          + revoke_rules_on_delete = false
          + tags                   = {
              + "Name" = "gitlab"
            }
          + tags_all               = {
              + "Name" = "gitlab"
            }
          + vpc_id                 = "vpc-02fefefefefe"

          + timeouts {}
        }

      # aws_volume_attachment.ebs_attachment_gitlab will be created
      + resource "aws_volume_attachment" "ebs_attachment_gitlab" {
          + device_name = "/dev/sdh"
          + id          = (known after apply)
          + instance_id = (known after apply)
          + volume_id   = (known after apply)
        }

    Plan: 4 to add, 0 to change, 0 to destroy.

    Do you want to perform these actions?
      Terraform will perform the actions described above.
      Only 'yes' will be accepted to approve.

      Enter a value: yes
```

输入 yes 之后，我们的 4 种资源都会被创建好了。

```
aws_instance.gitlab: Creating...
aws_instance.gitlab: Still creating... [10s elapsed]
aws_instance.gitlab: Creation complete after 13s [id=i-0afefefefe]
aws_volume_attachment.ebs_attachment_gitlab: Creating...
aws_volume_attachment.ebs_attachment_gitlab: Still creating... [10s elapsed]
aws_volume_attachment.ebs_attachment_gitlab: Still creating... [20s elapsed]
aws_volume_attachment.ebs_attachment_gitlab: Creation complete after 21s [id=vai-28fefefe]

Apply complete! Resources: 4 added, 0 changed, 0 destroyed.
```

# eip

下面我们给 gitlab 机器分配一个 `EIP`，它是一个 public ip 地址，我们可以通过这个 `EIP` 来访问 gitlab 机器。

EN: Add public ip for ec2 instance

```hcl
resource "aws_eip" "gitlab" {
  vpc = true
  tags = {
    Name = "gitlab"
  }
}

resource "aws_eip_association" "eip_association_gitlab" {
  instance_id   = aws_instance.gitlab.id
  allocation_id = aws_eip.gitlab.id
}
```

执行 `terraform plan` 后，我们会看到下面的输出：

```
# aws_eip.gitlab will be created
  + resource "aws_eip" "gitlab" {
      + allocation_id        = (known after apply)
      + association_id       = (known after apply)
      + carrier_ip           = (known after apply)
      + customer_owned_ip    = (known after apply)
      + domain               = (known after apply)
      + id                   = (known after apply)
      + instance             = (known after apply)
      + network_border_group = (known after apply)
      + network_interface    = (known after apply)
      + private_dns          = (known after apply)
      + private_ip           = (known after apply)
      + public_dns           = (known after apply)
      + public_ip            = (known after apply)
      + public_ipv4_pool     = (known after apply)
      + tags                 = {
          + "Name" = "gitlab"
        }
      + tags_all             = {
          + "Name" = "gitlab"
        }
      + vpc                  = true
    }

  # aws_eip_association.eip_association_gitlab will be created
  + resource "aws_eip_association" "eip_association_gitlab" {
      + allocation_id        = (known after apply)
      + id                   = (known after apply)
      + instance_id          = (known after apply)
      + network_interface_id = (known after apply)
      + private_ip_address   = (known after apply)
      + public_ip            = (known after apply)
    }
```

它会创建两个资源：一个 `EIP` 和一个 `EIP` 关联，这样，我们的 gitlab 机器就可以通过这个 `EIP` 访问了。

```
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_eip.gitlab will be created
  + resource "aws_eip" "gitlab" {
      + allocation_id        = (known after apply)
      + association_id       = (known after apply)
      + carrier_ip           = (known after apply)
      + customer_owned_ip    = (known after apply)
      + domain               = (known after apply)
      + id                   = (known after apply)
      + instance             = (known after apply)
      + network_border_group = (known after apply)
      + network_interface    = (known after apply)
      + private_dns          = (known after apply)
      + private_ip           = (known after apply)
      + public_dns           = (known after apply)
      + public_ip            = (known after apply)
      + public_ipv4_pool     = (known after apply)
      + tags                 = {
          + "Name" = "gitlab"
        }
      + tags_all             = {
          + "Name" = "gitlab"
        }
      + vpc                  = true
    }

  # aws_eip_association.eip_association_gitlab will be created
  + resource "aws_eip_association" "eip_association_gitlab" {
      + allocation_id        = (known after apply)
      + id                   = (known after apply)
      + instance_id          = "i-0afefefefe"
      + network_interface_id = (known after apply)
      + private_ip_address   = (known after apply)
      + public_ip            = (known after apply)
    }

Plan: 2 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

aws_eip.gitlab: Creating...
aws_eip.gitlab: Creation complete after 0s [id=eipalloc-ppp-kkk-0xff0xff]
aws_eip_association.eip_association_gitlab: Creating...
aws_eip_association.eip_association_gitlab: Creation complete after 1s [id=eipassoc-kukukulala]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

# gitlab setup

在结束了基础设施的创建之后，我们需要安装及设置 gitlab 软件本身。

我们使用 https://github.com/sameersbn/docker-gitlab 这个作为 `GitLab` 的镜像来安装 gitlab。

准备 `docker-compose.yml` 文件，运行 `docker-compose up -d` 命令，这样我们就可以看到 gitlab 在运行了。

```yaml
version: "2.3"

services:
  redis:
    restart: always
    image: redis:6.2.6
    command:
      - --loglevel warning
    volumes:
      - ./redis-data:/data:Z

  postgresql:
    restart: always
    image: sameersbn/postgresql:12-20200524
    volumes:
      - ./postgresql-data:/var/lib/postgresql:Z
    environment:
      - DB_USER=gitlab
      - DB_PASS=password
      - DB_NAME=gitlabhq_production
      - DB_EXTENSION=pg_trgm,btree_gist

  gitlab:
    restart: always
    image: sameersbn/gitlab:14.9.2
    depends_on:
      - redis
      - postgresql
    ports:
      - "10080:80"
      # - "443:443"
      - "10022:22"
    volumes:
      - ./gitlab-data:/home/git/data:Z
    healthcheck:
      test: ["CMD", "/usr/local/sbin/healthcheck"]
      interval: 5m
      timeout: 10s
      retries: 3
      start_period: 5m
    environment:
      - DEBUG=true

      - DB_ADAPTER=postgresql
      - DB_HOST=postgresql
      - DB_PORT=5432
      - DB_USER=gitlab
      - DB_PASS=password
      - DB_NAME=gitlabhq_production

      - REDIS_HOST=redis
      - REDIS_PORT=6379

      - GITLAB_HTTPS=false
        #- SSL_SELF_SIGNED=true

      - GITLAB_HOST=<eip ipv4 address>
        #- GITLAB_PORT=443
      - GITLAB_SSH_PORT=10022
      - GITLAB_RELATIVE_URL_ROOT=
      - GITLAB_SECRETS_DB_KEY_BASE=FF11111
      - GITLAB_SECRETS_SECRET_KEY_BASE=FF22222
      - GITLAB_SECRETS_OTP_KEY_BASE=FF33333

volumes:
  redis-data:
  postgresql-data:
  gitlab-data:
```

# GitLab clone

因为我们在 `docker-compose.yml` 中指定了 `10022:22` 这样的端口映射，在克隆代码时，所以我们需要指定端口信息:

```shell
git clone ssh://git@<public_ip>:10022/<username>/<repo>.git
```

# GitLab administration

`gitlab` 运行之后，我们可以对 `gitlab` 进行管理以增加其安全性，比如：

- 开启两步认证 `MFA`
- 禁用注册功能，只允许用户通过邮箱登录，取消勾选 `Menu > Admin > Settings > General > Sign-up restrictions > Sign-up enabled`
- 进入 `Menu > Admin > Settings > General > Visibility and access controls section`，限制 `Restricted visibility levels` 为 `public`，这样可以限制只有登录用户才能查看 user profile
- 取消用户注册，设置 `Sign-up restrictions` 为 `disabled`
- 限制两步认证才能登录，勾选 `Sign-in restrictions > Two-factor authentication > Enforce two-factor authentication`

好了，说到这里 `GitLab` 已经初步搭建完成，下面就可以自由的写 bug 啦。
