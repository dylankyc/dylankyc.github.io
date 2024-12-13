# Register gitlab runner on Amazon Linux 2

- [Intro](#intro)
- [Start AWS EC2 Instance](#start-aws-ec2-instance)
- [Install gitlab runner on Amazon Linux 2](#install-gitlab-runner-on-amazon-linux-2)
- [Register gitlab runner](#register-gitlab-runner)
- [Set GITLAB_PORT to 443](#set-gitlab_port-to-443)
- [Write .gitlab-ci.yml for go project](#write-gitlab-ciyml-for-go-project)
- [Test the runner](#test-the-runner)

# Intro

In this article, we will show you how to register gitlab runner on Amazon Linux 2 and setup gitlab ci for go project.

# Start AWS EC2 Instance

You can create aws ec2 instance either in aws console or using aws cli.

Notice you should enable public ip address if your aws ec2 instance is in public subnet of you VPC, otherwise you will not be able to access the internet!

# Install gitlab runner on Amazon Linux 2

Next, we'll install gitlab runner on Amazon Linux 2. Run the following bash script:

```bash
# install on amazon linux 2
# refs: https://github.com/beda-software/FAQ/blob/master/aws-ec2-gitlab-runner.md
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh" | sudo bash
sudo -E yum install gitlab-runner
sudo amazon-linux-extras install docker
sudo service docker start
sudo usermod -a -G docker ec2-user
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
sudo usermod -a -G docker ec2-user
sudo yum install -y git
```

# Register gitlab runner

You can register gitlab runner by using `gitlab-ci-multi-runner` command:

```bash
# register runner
# sudo gitlab-ci-multi-runner register -n --url GITLAB_URL --registration-token "TOKEN"   --executor docker   --description "Name of docker runner"   --docker-image "docker:latest" --docker-privileged
```

Replace `GITLAB_URL` and `TOKEN` with your self-hosted gitlab url and gitlab token. You can go to `https://gitlab.mycompany.com/admin/runners` and click `Register an instance runner` button, and you will see the `TOKEN` in the popup.

```bash
# register runner in gitlab ccc
sudo gitlab-ci-multi-runner register -n --url https://gitlab.mycompany.com/ --registration-token "A____TOKEN_____A"   --executor docker   --description "Name of docker runner"   --docker-image "docker:latest" --docker-privileged
```

Output:

```
Runtime platform                                    arch=amd64 os=linux pid=8095 revision=865283c5 version=16.1.0
Running in system-mode.

WARNING: Support for registration tokens and runner parameters in the 'register' command has been deprecated in GitLab Runner 15.6 and will be replaced with support for authentication tokens. For more information, see https://gitlab.com/gitlab-org/gitlab/-/issues/380872
Registering runner... succeeded                     runner=paaaaaaa
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!

Configuration (with the authentication token) was saved in "/etc/gitlab-runner/config.toml"
```

You will see a runner with name `paaaaaaa` is registered successfully.

And you can also config the runner in `/etc/gitlab-runner/config.toml`.

You can go to runners page in gitlab Admin and check if the runner is registered successfully.

# Set GITLAB_PORT to 443

As the runner is registered successfully, you may encounter an error like this:

```
gitlab-runner version 14.10.1 fails to clone a repo configured with a https repo URL, stating HTTP Basic: Access denied.
```

As we deploy gitlab using `https://github.com/sameersbn/docker-gitlab`, you may need to set `GITLAB_PORT` to `443` based on your configuration for load balancer above gitlab service, or add `clone_url` to the runner config file `/etc/gitlab-runner/config.toml`. Also, don't forget to restart docker servcie `sudo service docker restart`.

```toml
concurrent = 1
check_interval = 0
shutdown_timeout = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "Name of docker runner"
  url = "https://gitlab.mycompany.com/"
  clone_url = "https://gitlab.mycompany.com/"
  id = 1
  token = "s_____________n"
  token_obtained_at = 2023-07-22T01:20:41Z
  token_expires_at = 0001-01-01T00:00:00Z
  executor = "docker"
  [runners.cache]
    MaxUploadedArchiveSize = 0
  [runners.docker]
    tls_verify = false
    image = "docker:latest"
    privileged = true
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache"]
    shm_size = 0
```

# Write .gitlab-ci.yml for go project

For go project, you can write a gitlab ci file and test the runner.

Here is a simple gitlab ci file `.gitlab-ci.yml` for go project:

```yaml
# You can copy and paste this template into a new `.gitlab-ci.yml` file.
# You should not add this template to an existing `.gitlab-ci.yml` file by using the `include:` keyword.
#
# To contribute improvements to CI/CD templates, please follow the Development guide at:
# https://docs.gitlab.com/ee/development/cicd/templates.html
# This specific template is located at:
# https://gitlab.com/gitlab-org/gitlab/-/blob/master/lib/gitlab/ci/templates/Go.gitlab-ci.yml

image: golang:latest

stages:
  - test
  - build
  - deploy

format:
  stage: test
  script:
    - go fmt $(go list ./... | grep -v /vendor/)
    - go vet $(go list ./... | grep -v /vendor/)
    - go test -race $(go list ./... | grep -v /vendor/)

compile:
  stage: build
  script:
    - mkdir -p mybinaries
    - go build -o mybinaries ./...
  artifacts:
    paths:
      - mybinaries

deploy:
  stage: deploy
  script: echo "Define your deployment script!"
  environment: production
```

# Test the runner

You can trigger ci either by committing on `main` branch or clicking `Run Pipeline` in your project's pipeline page, i.e https://gitlab.mycompany.com/myname/go-ci-test/-/pipelines

If the pipeline is passed, your gitlab runner is runner successfully.
