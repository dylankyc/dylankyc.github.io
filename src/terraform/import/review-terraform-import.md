# Review terraform import

首先看一下 `terraform import` 的参数:

    The import command expects two arguments.
    Usage: terraform [global options] import [options] ADDR ID

      Import existing infrastructure into your Terraform state.

      This will find and import the specified resource into your Terraform
      state, allowing existing infrastructure to come under Terraform
      management without having to be initially created by Terraform.

      The ADDR specified is the address to import the resource to. Please
      see the documentation online for resource addresses. The ID is a
      resource-specific ID to identify that resource being imported. Please
      reference the documentation for the resource type you're importing to
      determine the ID syntax to use. It typically matches directly to the ID
      that the provider uses.

      The current implementation of Terraform import can only import resources
      into the state. It does not generate configuration. A future version of
      Terraform will also generate configuration.

      Because of this, prior to running terraform import it is necessary to write
      a resource configuration block for the resource manually, to which the
      imported object will be attached.

      This command will not modify your infrastructure, but it will make
      network requests to inspect parts of your infrastructure relevant to
      the resource being imported.

    Options:

      -config=path            Path to a directory of Terraform configuration files
                              to use to configure the provider. Defaults to pwd.
                              If no config files are present, they must be provided
                              via the input prompts or env vars.

      -allow-missing-config   Allow import when no resource configuration block exists.

      -input=false            Disable interactive input prompts.

      -lock=false             Don't hold a state lock during the operation. This is
                              dangerous if others might concurrently run commands
                              against the same workspace.

      -lock-timeout=0s        Duration to retry a state lock.

      -no-color               If specified, output won't contain any color.

      -var 'foo=bar'          Set a variable in the Terraform configuration. This
                              flag can be set multiple times. This is only useful
                              with the "-config" flag.

      -var-file=foo           Set variables in the Terraform configuration from
                              a file. If "terraform.tfvars" or any ".auto.tfvars"
                              files are present, they will be automatically loaded.

      -ignore-remote-version  A rare option used for the remote backend only. See
                              the remote backend documentation for more information.

      -state, state-out, and -backup are legacy options supported for the local
      backend only. For more information, see the local backend's documentation.

运行 `terraform import`，导入安全组（security group）：

    terraform import aws_security_group.gitlab sg-f2f2f2f2f2f2f2f2

    Error: resource address "aws_security_group.gitlab" does not exist in the configuration.

    Before importing this resource, please create its configuration in the root module. For example:

    resource "aws_security_group" "gitlab" {
      # (resource arguments)
    }

在导入资源的时候，提示错误，我们必须首先创建资源的配置，然后再导入资源。

在 `main.tf` 文件里添加以上资源 `resource "aws_security_group" "gitlab" {}`，再次运行 `terraform import` 命令:

    terraform import aws_security_group.gitlab sg-f2f2f2f2f2f2f2f2

输出:

    aws_security_group.gitlab: Importing from ID "sg-f2f2f2f2f2f2f2f2"...
    aws_security_group.gitlab: Import prepared!
      Prepared aws_security_group for import
    aws_security_group.gitlab: Refreshing state... [id=sg-f2f2f2f2f2f2f2f2]


    Import successful!

    The resources that were imported are shown above. These resources are now in
    your Terraform state and will henceforth be managed by Terraform.

import 之后，运行 `terraform plan` 来检查状态：

    aws_security_group.gitlab: Refreshing state... [id=sg-f2f2f2f2f2f2f2f2]

    Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following
    symbols:
    -/+ destroy and then create replacement

    Terraform will perform the following actions:

      # aws_security_group.gitlab must be replaced
    -/+ resource "aws_security_group" "gitlab" {
          ~ arn                    = "arn:aws:ec2:us-east-1:81818181888:security-group/sg-f2f2f2f2f2f2f2f2" -> (known after app
    ly)
          ~ description            = "Security group for gitlab" -> "Managed by Terraform" # forces replacement
          ~ egress                 = [
              - {
                  - cidr_blocks      = [
                      - "0.0.0.0/0",
                    ]
                  - description      = ""
                  - from_port        = 0
                  - ipv6_cidr_blocks = []
                  - prefix_list_ids  = []
                  - protocol         = "-1"
                  - security_groups  = []
                  - self             = false
                  - to_port          = 0
                },
            ] -> (known after apply)
          ~ id                     = "sg-f2f2f2f2f2f2f2f2" -> (known after apply)
          ~ ingress                = [
              - {
                  - cidr_blocks      = [
                      - "55.188.0.0/16",
                    ]
                  - description      = ""
                  - from_port        = 22
                  - ipv6_cidr_blocks = []
                  - prefix_list_ids  = []
                  - protocol         = "tcp"
                  - security_groups  = []
                  - self             = false
                  - to_port          = 22
                },
              - {
                  - cidr_blocks      = []
                  - description      = "r2r2-v2"
                  - from_port        = 10080
                  - ipv6_cidr_blocks = []
                  - prefix_list_ids  = []
                  - protocol         = "tcp"
                  - security_groups  = [
                      - "sg-0x1231238888",
                    ]
                  - self             = false
                  - to_port          = 10080
                },
            ] -> (known after apply)
          ~ name                   = "gitlab" -> (known after apply)
          + name_prefix            = (known after apply)
          ~ owner_id               = "81818181888" -> (known after apply)
          + revoke_rules_on_delete = false
          - tags                   = {
              - "Name" = "gitlab"
            } -> null
          ~ tags_all               = {
              - "Name" = "gitlab"
            } -> (known after apply)
          ~ vpc_id                 = "vpc-0102030401020304" -> (known after apply)

          - timeouts {}
        }

    Plan: 1 to add, 0 to change, 1 to destroy.

    Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run
    "terraform apply" now.

可以看到，我们导入的资源没有在 `main.tf` 配置文件中，如果现在我们运行 `terraform apply` 则会删除这些资源。

为了让 `terraform.tfstate` 和 cloud provider 的配置一致，需要手动修改 tf 文件，添加对 import 资源的配置，添加的方法，基本上将 `terraform plan` 的输出（注意去掉 `-` `~` 这些）拷贝到对应的 resource 中，并做一些调整。对于 security group，添加资源配置的方法如下：

    resource "aws_security_group" "gitlab" {
      # arn                    = "arn:aws:ec2:us-east-1:81818181888:security-group/sg-f2f2f2f2f2f2f2f2" -> (known after apply)
      description = "Security group for gitlab"
      egress = [
        {
          cidr_blocks = [
            "0.0.0.0/0",
          ]
          description      = ""
          from_port        = 0
          ipv6_cidr_blocks = []
          prefix_list_ids  = []
          protocol         = "-1"
          security_groups  = []
          self             = false
          to_port          = 0
        },
      ]
      # id                     = "sg-f2f2f2f2f2f2f2f2" -> (known after apply)
      ingress = [
        {
          cidr_blocks = [
            "55.188.0.0/16",
          ]
          description      = ""
          from_port        = 22
          ipv6_cidr_blocks = []
          prefix_list_ids  = []
          protocol         = "tcp"
          security_groups  = []
          self             = false
          to_port          = 22
        },
        {
          cidr_blocks      = []
          description      = "r2r2-v2"
          from_port        = 10080
          ipv6_cidr_blocks = []
          prefix_list_ids  = []
          protocol         = "tcp"
          security_groups = [
            "sg-0x1231238888",
          ]
          self    = false
          to_port = 10080
        },
      ]
      name = "gitlab"
      # owner_id               = "81818181888" -> (known after apply)
      revoke_rules_on_delete = false
      tags = {
        "Name" = "gitlab"
      }
      tags_all = {
        "Name" = "gitlab"
      }
      vpc_id = "vpc-0102030401020304"

      timeouts {}
    }

首先 `terraform init`:

    Initializing the backend...

    Initializing provider plugins...
    - Finding latest version of hashicorp/aws...
    - Installing hashicorp/aws v4.1.0...
    - Installed hashicorp/aws v4.1.0 (signed by HashiCorp)

    Terraform has created a lock file .terraform.lock.hcl to record the provider
    selections it made above. Include this file in your version control repository
    so that Terraform can guarantee to make the same selections by default when
    you run "terraform init" in the future.

    Terraform has been successfully initialized!

    You may now begin working with Terraform. Try running "terraform plan" to see
    any changes that are required for your infrastructure. All Terraform commands
    should now work.

    If you ever set or change modules or backend configuration for Terraform,
    rerun this command to reinitialize your working directory. If you forget, other
    commands will detect it and remind you to do so if necessary.

# refs

https://www.terraform.io/cli/import/usage
