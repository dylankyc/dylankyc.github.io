# 一个 s3 bucket 分配权限的 terraform 管理介绍.md

- [Intro](#intro)
  - [第一步](#%E7%AC%AC%E4%B8%80%E6%AD%A5)
  - [第二步](#%E7%AC%AC%E4%BA%8C%E6%AD%A5)
  - [第三步](#%E7%AC%AC%E4%B8%89%E6%AD%A5)
  - [第四步](#%E7%AC%AC%E5%9B%9B%E6%AD%A5)
  - [测试](#%E6%B5%8B%E8%AF%95)
- [摘要](#%E6%91%98%E8%A6%81)
- [Refs](#refs)

# Intro

作为一个 S3 的管理员，一个很常见的场景是，需要对分配给某个用户对某个 bucket 写操作的权限。

针对 bucket 的授权，我们可以使用 bucket policy 或 user policy 或同时使用二者，此时我们采取同时使用二者的方案。

这里涉及到以下几方面：

- IAM User：权限的被授予者，AWS account 下的普通用户
- AWS Account Administrator：权限的授予者，是通过根凭证（Root credentials）创建的具有 Admin 权限的管理员用户
- S3 bucket：需要一定权限访问的 S3 bucket
- bucket policy：附加到 bucket 上的策略
- user policy：附加给用户的策略

操作流程如下，其中 Root credentials 创建 AWS Account Administrator 已省略：

- 1、AWS Account Administrator 创建 bucket（如果测试 bucket 已存在，这步可以省略）
- 2、AWS Account Administrator 创建 IAM User，用户名 testuser，记录 credentials 和 arn
- 3、给测试 bucket 附加 bucket policy，允许 testuser 拥有 `s3:GetBucketLocation` 和 `s3:ListBucket` 两个权限，另外允许对 bucket 下子路径 `s3:GetObject` 的权限
- 4、给 testuser 附加对 bucket `s3:GetObject` 的权限，可以上传文件

## 第一步

创建 bucket 比较简单，我们在 aws 控制台简单创建了一个用于测试目的的 bucket 叫 `k8s-cluster-bucket-test`。

## 第二步

创建一个 users module，方便模块化管理。

`variables.tf` 文件定义了 `module` 的参数：

    variable "username" {
      default = ""
    }

`outputs.tf` 定义了模块的输出，为了给 iam user 附加 policy，此时我们需要输出 iam user 的 `arn`。

    output "user_arn" {
      value = "${aws_iam_user.user.arn}"
    }

如果把 `module` 看作一个函数的话，`user_arn` 这个 `output` 类似 `module` 的返回值，它将在给 bucket 设置 policy 及 iam user 设置 policy 时都会用到。

利用 `aws_iam_user` 来创建 iam user：

`main.tf`

    resource "aws_iam_user" "user" { name = "${var.username}" }

我们可以利用这个 `module` 创建多个符合特定条件的 user，模块化能让代码结构更加清晰。如创建一个叫 testuser 的 iam user 可以这么做：

    module "testuser" {
        source="./users"
        username = "testuser"
    }

## 第三步

因为 bucket policy 需要知晓 iam user 的 arn，所以我们在 `variables.tf` 做了定义：

`variables.tf`

    variable "user_arn" {
      default = ""
    }

之后我们使用 `aws_iam_policy_document` 来定义附加给 bucket 的 policy。

`aws_iam_policy_document` 是一个 `data source`，可以理解为它会生成我们需要的 bucket policy。

为了简化，我们 hardcode 了 bucket 的 arn，当然更好的做法是把它作为变量抽象出来，也就是放到 `module` 的 `variables.tf` 里
。

    data "aws_iam_policy_document" "s3-bucket-policy-document" {
        statement {
            effect  = "Allow"
            actions = [
                "s3:GetBucketLocation",
                "s3:ListBucket"
            ]
            principals {
                type = "AWS"
                identifiers = [
                  "${var.user_arn}"
                ]
            }
            resources = [
                "arn:aws-cn:s3:::k8s-cluster-bucket-test"
            ]
        }

        statement {
            effect  = "Allow"
            actions = [
                "s3:GetObject"
            ]
            principals {
                type = "AWS"
                identifiers = [
                  "${var.user_arn}"
                ]
            }
            resources = [
                "arn:aws-cn:s3:::k8s-cluster-bucket-test/*"
            ]
        }
    }

这里包含两个 statement，如上所述，第一个 statement 允许 testuser 拥有 `s3:GetBucketLocation` 和 `s3:ListBucket` 两个权限，第二个 statement 允许对 bucket 下子路径拥有 `s3:GetObject` 的权限。

> Principal 是权限的委托人，此时的例子用来给 IAM User 访问 s3 bucket resource 的权限。除此之外，委托人还包括很多类型如特定 AWS 账户、单个或多个 iam user、iam role 或 aws 的服务，具体可参考 Principal 的官方文档。

利用 `aws_s3_bucket_policy` 定义资源，这将给我们的目标 bucket 附加 policy。

    resource "aws_s3_bucket_policy" "k8s-cluster-bucket-test" {
      bucket = "k8s-cluster-bucket-test"
      policy = "${data.aws_iam_policy_document.s3-bucket-policy-document.json}"
    }

## 第四步

回到 users module。

我们利用 `aws_iam_policy` 来创建针对指定 bucket 的写权限 `s3:PutObject`。

    resource "aws_iam_policy" "s3-put-object-policy" {
      name        = "s3-put-object-policy"
      description = "A s3 pub object policy"
      policy      = <<EOF
    {
       "Version": "2012-10-17",
       "Statement": [
          {
             "Sid": "PermissionForObjectOperations",
             "Effect": "Allow",
             "Action": [
                "s3:PutObject"
             ],
             "Resource": [
                "arn:aws-cn:s3:::k8s-cluster-bucket-test/*"
             ]
          }
       ]
    }
    EOF
    }

然后通过 `aws_iam_user_policy_attachment` 来给新创建的 iam user 附加 policy。

    resource "aws_iam_user_policy_attachment" "s3-put-object-policy-attach" {
      user       = "${aws_iam_user.user.name}"
      policy_arn = "${aws_iam_policy.s3-put-object-policy.arn}"
    }

## 测试

需要在 aws credentials 里使用 testuser 的 credentials 来指定用 testuser 运行 aws 命令。

当我们没有开放权限时（未添加第四步的配置），上传文件会出错：

    $ aws s3api put-object --bucket k8s-cluster-bucket-test --key 1.png --body 1.png

    An error occurred (AccessDenied) when calling the PutObject operation: Access Denied

添加了对指定 bucket 的 PutObject 权限之后，上传成功了（It works!）。

    $ aws s3api put-object --bucket k8s-cluster-bucket-test --key 1.png --body 1.png
    {
        "ETag": "\"c4df98a9588d3baba05aaa9212e2fe37\""
    }

题图 lensinkmitchel from unsplash

# 摘要

作为一个 aws 的管理员，只需要一个技能让你告别 aws 控制台

# Refs

https://docs.aws.amazon.com/AmazonS3/latest/dev/example-walkthroughs-managing-access-example1.html

参考：
https://docs.aws.amazon.com/AmazonS3/latest/dev/example-walkthroughs-managing-access.html

更复杂一点的例子：

为 S3 存储桶中的对象授予公共读取访问权限
https://amazonaws-china.com/cn/premiumsupport/knowledge-center/read-access-objects-s3-bucket/

如何将亚马逊 AWS S3 存储桶的访问权限到一个特定 IAM 角色 | 亚马逊 AWS 官方博客
https://amazonaws-china.com/cn/blogs/china/securityhow-to-restrict-amazon-s3-bucket-access-to-a-specific-iam-role/

How to Restrict Amazon S3 Bucket Access to a Specific IAM Role | AWS Security Blog
https://amazonaws-china.com/cn/blogs/security/how-to-restrict-amazon-s3-bucket-access-to-a-specific-iam-role/
