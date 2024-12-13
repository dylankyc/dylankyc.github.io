# Build multi-arch docker image using buildx

- [Setup simple golang project using go mod](#setup-simple-golang-project-using-go-mod)
- [.gitlab-ci.yml](#gitlab-ciyml)
- [Setup CI variable to authenticate gitlab to aws ecr](#setup-ci-variable-to-authenticate-gitlab-to-aws-ecr)
- [Run ci](#run-ci)
- [Inspect the image](#inspect-the-image)

# Setup simple golang project using go mod

Iniit golang project using `go mod`:

```bash
go mod init gitlab.com/aoaojiaoaoaojiao/go-multi-arch
```

Write `main.go`:

```go
package main

import "fmt"

func main() {
	fmt.Println("vim-go")
}
```

# .gitlab-ci.yml

```yaml
# You can override the included template(s) by including variable overrides
# SAST customization: https://docs.gitlab.com/ee/user/application_security/sast/#customizing-the-sast-settings
# Secret Detection customization: https://docs.gitlab.com/ee/user/application_security/secret_detection/#customizing-settings
# Dependency Scanning customization: https://docs.gitlab.com/ee/user/application_security/dependency_scanning/#customizing-the-dependency-scanning-settings Container Scanning customization: https://docs.gitlab.com/ee/user/application_security/container_scanning/#customizing-the-container-scanning-settings
# Note that environment variables can be set in several places
# See https://docs.gitlab.com/ee/ci/variables/#cicd-variable-precedence
# stages:
# - test
# sast:
#   stage: test
# include:
# - template: Security/SAST.gitlab-ci.yml

image: docker:20.10.8

stages:
  - build-push

variables:
  DOCKER_DRIVER: overlay2
  BUILDX_VERSION: "v0.6.1"
  BUILDX_ARCH: "linux-amd64"
  AWS_DEFAULT_REGION: us-east-1
  AWS_ECR_NAME: 444333555686.dkr.ecr.us-east-1.amazonaws.com/orders/orders
  AWS_ACCOUNT_ID: 444333555686
  #DOCKER_IMAGE_NAME: your-docker-image-name
  #DOCKER_USERNAME: AWS
  GIT_COMMIT_SHA: ${CI_COMMIT_SHORT_SHA}

build and push:
  stage:
    build-push
    #image: docker:dind
  image: docker:20.10.8-dind
  services:
    - docker:dind
  before_script:
    - apk update
    - apk add --no-cache curl python3 py3-pip git
    - pip3 install awscli
    - wget -O /usr/bin/docker-buildx https://github.com/docker/buildx/releases/download/${BUILDX_VERSION}/buildx-${BUILDX_VERSION}.${BUILDX_ARCH}
    - chmod +x /usr/bin/docker-buildx
    - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
  script:
    #- docker buildx create --use
    #- docker buildx build --platform linux/amd64,linux/arm64 -t $AWS_ECR_NAME:$GIT_COMMIT_SHA --push .
    - docker-buildx create --use
    - docker-buildx build --platform linux/amd64,linux/arm64 -t $AWS_ECR_NAME:$GIT_COMMIT_SHA --push .
```

# Setup CI variable to authenticate gitlab to aws ecr

In your GitLab project, go to `Settings` > `CI/CD`. Set the following CI/CD variables:

Environment variable name Value

- `AWS_ACCESS_KEY_ID` Your Access key ID.
- `AWS_SECRET_ACCESS_KEY` Your secret access key.
- `AWS_DEFAULT_REGION` Your region code. You might want to confirm that the AWS service you intend to use is available in the chosen region.

Variables are protected by default. To use GitLab CI/CD with branches or tags that are not protected, clear the Protect variable checkbox.

# Run ci

Any time you push code into the repo, gitlab runn run ci pipieline and images with two platforms will be push to AWS ECR, based on the arguments of `docker-buildx build`: `--platform linux/amd64,linux/arm64`

# Inspect the image

Inspect the image:

```bash
docker manifest inspect 444333555686.dkr.ecr.us-east-1.amazonaws.com/orders/orders:71db070c
```

Output:

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
  "manifests": [
    {
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "size": 1682,
      "digest": "sha256:65666a6rbcccc7er7we7w7238238d7ds7fd7sdfs7fs7ds7s7s7",
      "platform": {
        "architecture": "amd64",
        "os": "linux"
      }
    },
    {
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "size": 1681,
      "digest": "sha256:i23iwejfdsadfasjdsapfsdaphfdasusdahudshisadhoasdodshd",
      "platform": {
        "architecture": "arm64",
        "os": "linux"
      }
    }
  ]
}
```
