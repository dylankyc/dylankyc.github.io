# Install gitlab runner on ubuntu

- [Intro](#intro)
  - [Check GPU info on instance](#check-gpu-info-on-instance)
  - [Install gitlab runner](#install-gitlab-runner)
  - [Register gitlab runner](#register-gitlab-runner)
  - [Uninstall GitLab Runner](#uninstall-gitlab-runner)
- [Start gitlab-runner in container](#start-gitlab-runner-in-container)
  - [Gitlab runner token](#gitlab-runner-token)
  - [Use local volume](#use-local-volume)
  - [docker volume](#docker-volume)
- [Register gitlab runner in non-interactive mode](#register-gitlab-runner-in-non-interactive-mode)
- [stop gitlab-runner](#stop-gitlab-runner)
- [Refs](#refs)

# Intro

I've been using GitLab CI/CD for a while now, and I have to say that it's an amazing tool for code management and automating builds and deployments.

In this blog post, I'll share my experience installing and using GitLab Runner on Ubuntu with GPU instance.

## Check GPU info on instance

First, check the GPU info on the instance:

```bash
(base) smolai@smolai-Z790-UD-AX:~$ nvidia-smi
Mon Sep 18 15:57:06 2023
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 525.116.04   Driver Version: 525.116.04   CUDA Version: 12.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA GeForce ...  Off  | 00000000:01:00.0 Off |                  Off |
|  0%   35C    P8     5W / 450W |   3754MiB / 24564MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|    0   N/A  N/A     13290      C   python                           3752MiB |
+-----------------------------------------------------------------------------+
```

## Install gitlab runner

Next, we can install gitlab runner using the following command:

```bash
wget -qO - https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt install -y gitlab-runner
```

## Register gitlab runner

After installation, we can register gitlab runner using the following command:

```bash
# register gitlab runner
sudo gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.planetsmol.com" \
  --registration-token "glrt-ooooooiiiiiiiiii" \
  --description "docker-runner" \
  --executor "docker" \
  --docker-image ubuntu:latest
```

Here is the output of the command:

```bash
Runtime platform                                    arch=amd64 os=linux pid=74044 revision=f5dfa4d1 version=16.3.1
Running in system-mode.

Verifying runner... is valid                        runner=eooovvviii
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!

Configuration (with the authentication token) was saved in "/etc/gitlab-runner/config.toml"
root@smolai:/tmp# docker images
REPOSITORY                                                          TAG               IMAGE ID       CREATED          SIZE
registry.gitlab.com/gitlab-org/gitlab-runner/gitlab-runner-helper   x86_64-f5dfa4d1   4af7e8dd8eb7   18 seconds ago   64.1MB
meet                                                                latest            7d62cb955a7f   5 weeks ago      915MB
busybox                                                             latest            a416a98b71e2   2 months ago     4.26MB
```

## Uninstall GitLab Runner

If you want to completely remove GitLab Runner, run the following command:

```bash
# If you want to completely remove GitLab Runner, run the following command:
sudo apt purge --autoremove -y gitlab-runner

# Remove GPG key and repository:
sudo apt-key del 513111FF
sudo rm -rf /etc/apt/sources.list.d/runner_gitlab-runner.list

# Remove GitLab Runner user:
sudo deluser --remove-home gitlab-runner

#You can also remove GitLab Runner configuration:
sudo rm -rf /etc/gitlab-runner
```

# Start gitlab-runner in container

You have two options to start gitlab-runner in container.

To store gitlab-runner config in docker volume, you can either use docker volume or use local system volume.

After install gitlab-runner, you can start gitlab-runner in container.

You have to register gitlab runner after start gitlab-runner in container.

As the docs said:

> Runner registration is the process that links the runner with one or more GitLab instances. You must register the runner so that it can pick up jobs from the GitLab instance.

## Gitlab runner token

You have to obtain gitlab runner token from gitlab to register the runner.

Here is how to obtain the token from gitlab:

1. Login to gitlab
2. Click on the "Runners" button
3. Click on the "Tokens" button
4. Click on the "Create token" button
5. Copy the token

Notice, the gitlab runner authentication tokens have the prefix `glrt-`.

## Use local volume

Let's see how to use local system volume mounts to start the Runner container.

```bash
# Create the directory to mount the docker volume
mkdir -p /srv/gitlab-runner/config

# Start the GitLab Runner container
docker run -d --name gitlab-runner --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
```

Now the runner is started, let's see how to register it.

```bash
docker run --rm -it -v /srv/gitlab-runner/config:/etc/gitlab-runner gitlab/gitlab-runner register
```

## docker volume

If you want to use docker volume to start the Runner container, you can use the following command:

```bash
# Create the Docker volume
docker volume create gitlab-runner-config

# Start the GitLab Runner container using the volume we just created
docker run -d --name gitlab-runner --restart always \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v gitlab-runner-config:/etc/gitlab-runner \
    gitlab/gitlab-runner:latest
```

# Register gitlab runner in non-interactive mode

You can use non-interactive mode to register the runner, refer to https://docs.gitlab.com/runner/commands/index.html#non-interactive-registration for more details.

If you want to register the runner on linux, you can use the following command:

```bash
sudo gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.com/" \
  --token "$RUNNER_TOKEN" \
  --executor "docker" \
  --docker-image alpine:latest \
  --description "docker-runner"
```

If you want to register the runner through docker, you can use the following command:

```bash
docker run --rm -v /srv/gitlab-runner/config:/etc/gitlab-runner gitlab/gitlab-runner register \
  --non-interactive \
  --executor "docker" \
  --docker-image alpine:latest \
  --url "https://gitlab.com/" \
  --token "$RUNNER_TOKEN" \
  --description "docker-runner"
```

# stop gitlab-runner

To stop the gitlab-runner container, you can use the following command:

```bash
docker stop gitlab-runner && docker rm gitlab-runner
```

# Refs

Install
https://lindevs.com/install-gitlab-runner-on-ubuntu

Install gitlab runner
https://docs.gitlab.com/runner/install/docker.html

Another install doc
https://docs.gitlab.com/runner/install/linux-repository.html

Register runner
https://docs.gitlab.com/runner/register/index.html#docker
