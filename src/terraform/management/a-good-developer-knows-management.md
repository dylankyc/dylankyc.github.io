# A Good Developer knows management

没有不好的管理者，只有懒的管理者。

先看下我们在哪里：

    $pwd
    /home/fantasticmachine/terraform

像我们写 Hello world 的时候，我们需要一个 main 文件，另外的变量文件 variables.tf 体现了编程语言里的模块化思想：

    $tree
    .
    ├── main.tf
    └── variables.tf

`variables.tf` 定义了我们需要的变量，如 `access_key` `secret_key` 和 `region`

    $cat variables.tf
    variable "access_key" {}
    variable "secret_key" {}

    variable "region" {
        default = "cn-north-1"
    }

为了让 `terraform` 访问 `aws`，需要在 `main` 文件里加入 `iam` 配置，即加入 `access_key` 和 `secret_key`，配置信息需要放在 `terraform` 的 `aws` provider 里：

    $cat main.tf
    provider "aws" {
        access_key = "${var.access_key}"
        secret_key = "${var.secret_key}"
        region = "${var.region}"
    }

为了让 `terraform` 顺利连接 `aws`，我们需要在 `bash` 的环境变量里指定 `access_key` 和 `secret_key`:

    export TF_VAR_access_key="AKIAPLONGTIMEAGO"
    export TF_VAR_secret_key="Lukeskywalkershowmehowtousesword"
    export AWS_ACCESS_KEY=$TF_VAR_access_key
    export AWS_SECRET_KEY=$TF_VAR_secret_key
    export EC2_REGION=cn-north-1

好了，先拿 iam 开刀。

在控制台已经建了几个 group，如 Developer 等，我们现在让 `terraform` 管理。

导入到 main.tf 配置中：

    $terraform import aws_iam_group.Developers developers
    Error: resource address "aws_iam_group.Developers" does not exist in the configuration.

    Before importing this resource, please create its configuration in the root module. For example:

    resource "aws_iam_group" "Developers" {
    # (resource arguments)
    }

发现出错了！

错误信息说 `aws_iam_group.Developers` 这个资源不在配置文件中，需要我们手动创建。

    $cat >> main.tf <<EOF
    resource "aws_iam_group" "Developers" {
        # (resource arguments)
    }
    EOF

按照提示做好之后，再次运行 `terraform import`：

    $terraform import aws_iam_group.Developers developers
    Plugin reinitialization required. Please run "terraform init".
    Reason: Could not satisfy plugin requirements.

    Plugins are external binaries that Terraform uses to access and manipulate
    resources. The configuration provided requires plugins which can't be located,
    don't satisfy the version constraints, or are otherwise incompatible.

    1 error(s) occurred:

    * provider.aws: no suitable version installed
    version requirements: "(any version)"
    versions installed: none

    Terraform automatically discovers provider requirements from your
    configuration, including providers used in child modules. To see the
    requirements and constraints from each module, run "terraform providers".

    error satisfying plugin requirements

还是出错，再耐心看下出错信息：plugin 需要重新初始化，我们还要再运行 `terraform init`：

    $terraform init

    Initializing provider plugins...
    - Checking for available provider plugins on https://releases.hashicorp.com...
    - Downloading plugin for provider "aws" (2.3.0)...

    The following providers do not have any version constraints in configuration,
    so the latest version was installed.

    To prevent automatic upgrades to new major versions that may contain breaking
    changes, it is recommended to add version = "..." constraints to the
    corresponding provider blocks in configuration, with the constraint strings
    suggested below.

    * provider.aws: version = "~> 2.3"

    Terraform has been successfully initialized!

    You may now begin working with Terraform. Try running "terraform plan" to see
    any changes that are required for your infrastructure. All Terraform commands
    should now work.

    If you ever set or change modules or backend configuration for Terraform,
    rerun this command to reinitialize your working directory. If you forget, other
    commands will detect it and remind you to do so if necessary.

此时插件下载完毕，同时还提示在配置文件中加入 `provider.aws: version = "~> 2.3"` 来锁版本，防止大版本升级带来的 `breaking changes`，接触包管理工具，这点我们早已习以为常。

    sed -i '/provider "aws" {/a \ \ version = "~> 2.3"' main.tf

那我们再来导入下吧。

    $terraform import aws_iam_group.Developers developers
    aws_iam_group.Developers: Importing from ID "developers"...
    aws_iam_group.Developers: Import complete!
    Imported aws_iam_group (ID: developers)
    aws_iam_group.Developers: Refreshing state... (ID: developers)

    Import successful!

    The resources that were imported are shown above. These resources are now in
    your Terraform state and will henceforth be managed by Terraform.

功夫不负有心人，终于成功了！

    $terraform import aws_iam_group.Developers developers
    aws_iam_group.Developers: Importing from ID "developers"...
    aws_iam_group.Developers: Import complete!
    Imported aws_iam_group (ID: developers)
    aws_iam_group.Developers: Refreshing state... (ID: developers)

    Import successful!

    The resources that were imported are shown above. These resources are now in
    your Terraform state and will henceforth be managed by Terraform.

看看我们的 `terraform.tfstate` 状态吧。

    $cat terraform.tfstate
    {
        "version": 3,
        "terraform_version": "0.11.3",
        "serial": 1,
        "lineage": "95330000-1111-2222-3333-444455556666",
        "modules": [
            {
                "path": [
                    "root"
                ],
                "outputs": {},
                "resources": {
                    "aws_iam_group.Developers": {
                        "type": "aws_iam_group",
                        "depends_on": [],
                        "primary": {
                            "id": "developers",
                            "attributes": {
                                "arn": "arn:aws-cn:iam::311112222333:group/Developers",
                                "id": "developers",
                                "name": "Developers",
                                "path": "/",
                                "unique_id": "AGPUNIQUESLASHID"
                            },
                            "meta": {},
                            "tainted": false
                        },
                        "deposed": [],
                        "provider": "provider.aws"
                    }
                },
                "depends_on": []
            }
        ]
    }

`terraform.tfstate` 是集群真实状态的一个映射，也就是说，当运行 `terraform apply` 之后，集群的真实状态和本地配置和 `terraform.tfstate` 三者是等同的。

接下来，我们可以照猫画虎来导入其他 groups、roles 和 users 等，这样 iam 就可以统一用 terraform 来管理了，我们甚至能管理 ec2、security group 甚至整个 vpc。这样的好处不言而喻，我们可以随时监听真实世界的一草一木，因为 `terraform plan` 会很容易发现真实世界和本地配置的差异，任何对 iam 的修改都已在我们的掌控之中。

题图

https://unsplash.com/@leonelfdez
