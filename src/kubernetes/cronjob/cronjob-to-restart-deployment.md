# CronJob To Restart Deployment

- [CronJob To Restart Deployment](#cronjob-to-restart-deployment)
  - [Explain line by line](#explain-line-by-line)
    - [ServiceAccount](#serviceaccount)
    - [Role](#role)
    - [RoleBinding](#rolebinding)
    - [CronJob](#cronjob)

# CronJob To Restart Deployment

Here is an example of how to restart deployment using cronjob.

```yaml
---
# Service account the client will use to reset the deployment,
# by default the pods running inside the cluster can do no such things.
kind: ServiceAccount
apiVersion: v1
metadata:
  name: deployment-restart
  namespace: default
---
# allow getting status and patching only the one deployment you want to restart
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-restart
  namespace: default
rules:
  - apiGroups: ["apps", "extensions"]
    resources: ["deployments"]
    resourceNames: ["my-fast-and-robust-service"]
    verbs:
      # "list" and "watch" are only needed if you want to use `rollout status`
      ["get", "patch", "list", "watch"]
---
# bind the role to the service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployment-restart
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: deployment-restart
subjects:
  - kind: ServiceAccount
    name: deployment-restart
    namespace: default
---
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: deployment-restart
  namespace: default
spec:
  concurrencyPolicy: Forbid
  # cron spec of time, here, 8 o'clock
  schedule: "0 1 * * *"
  jobTemplate:
    spec:
      backoffLimit:
        # this has very low chance of failing, as all this does
        # is prompt kubernetes to schedule new replica set for
        # the deployment
        2
      activeDeadlineSeconds:
        # timeout, makes most sense with "waiting for rollout" variant specified below
        600
      template:
        spec:
          serviceAccountName:
            # name of the service account configured above
            deployment-restart
          restartPolicy: Never
          containers:
            - name: kubectl
              image:
                # probably any kubectl image will do,
                # optionaly specify version, but this
                # should not be necessary, as long the
                # version of kubectl is new enough to
                # have `rollout restart`
                bitnami/kubectl
              command:
                - "kubectl"
                - "rollout"
                - "restart"
                - "deployment/my-fast-and-robust-service"
```

这是一份 Kubernetes YAML 配置文件，用于创建一个 CronJob，定期重启指定名称的 Deployment。以下是配置文件的解释：

1. 通过创建 ServiceAccount，确保 Kubernetes 能够在崩溃时自动重启 Deployment。

2. 创建一个 RBAC Role，允许使用 `get`、`patch`、`list` 和 `watch` 操作，但仅限于指定的 Deployment（即 `my-fast-and-robust-service`）。

3. 创建一个 RBAC RoleBinding，将上述角色与先前创建的 ServiceAccount 绑定。

4. 创建一个 CronJob，定义了要执行的任务和运行频率。该任务将定期启动一个 Pod，并使用 kubectl 命令重启指定的 Deployment。

总之，这个配置文件实现了一个可以定期重启指定 `Deployment` 的 `CronJob`，以确保 `Deployment` 在运行过程中保持稳定性和可用性。这可能会有助于解决某些资源耗尽或性能问题，以及其他可能导致 `Deployment` 失败的情况。请注意，这应该被视为一种应急措施，应该在确定没有更好的解决方案之前谨慎使用。

## Explain line by line

### ServiceAccount

好的，下面是这个 YAML 配置文件中每一行的解释：

```
---
# Service account the client will use to reset the deployment,
# by default the pods running inside the cluster can do no such things.
kind: ServiceAccount
apiVersion: v1
metadata:
  name: deployment-restart
  namespace: default
```

- `---`：指示 Kubernetes YAML 文件的开始。

- `# Service account the client will use to reset the deployment,`：注释，提供了正在创建的 ServiceAccount 的描述。

- `kind: ServiceAccount`：指定 Kubernetes 对象的类型，这里是 ServiceAccount。

- `apiVersion: v1`：指定 API 版本，这里是 v1。

- `metadata:`：定义 Kubernetes 对象的元数据，包括名称和命名空间。

- `name: deployment-restart`：指定创建的 ServiceAccount 的名称为 deployment-restart。

- `namespace: default`：指定命名空间为 default。

### Role

```
---
# allow getting status and patching only the one deployment you want to restart
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-restart
  namespace: default
rules:
  - apiGroups: ["apps", "extensions"]
    resources: ["deployments"]
    resourceNames: ["my-fast-and-robust-service"]
    verbs:
      # "list" and "watch" are only needed if you want to use `rollout status`
      ["get", "patch", "list", "watch"]
```

- `---`：指示 Kubernetes YAML 文件的开始。

- `# allow getting status and patching only the one deployment you want to restart`：注释，提供了所创建的 Role 的描述。

- `apiVersion: rbac.authorization.k8s.io/v1`：指定 RBAC API 版本，这里是 v1。

- `kind: Role`：指定 Kubernetes 对象类型为 Role。

- `metadata:`：定义 Kubernetes 对象的元数据，包括名称和命名空间。

- `name: deployment-restart`：指定创建的 Role 的名称为 deployment-restart。

- `namespace: default`：指定命名空间为 default。

- `rules:`：定义权限规则。

- `- apiGroups: ["apps", "extensions"]`：指定 API 组。这里指定了 apps 和 extensions。

- `resources: ["deployments"]`：指定资源种类，这里是 deployments。

- `resourceNames: ["my-fast-and-robust-service"]`：指定资源名称，这里是 my-fast-and-robust-service。

- `verbs: ["get", "patch", "list", "watch"]`：指定允许执行的操作，这里包括 get、patch、list 和 watch。

### RoleBinding

```
---
# bind the role to the service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployment-restart
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: deployment-restart
subjects:
  - kind: ServiceAccount
    name: deployment-restart
    namespace: default
```

- `---`：指示 Kubernetes YAML 文件的开始。

- `# bind the role to the service account`：注释，提供了 RoleBinding 的描述。

- `apiVersion: rbac.authorization.k8s.io/v1`：指定 RBAC API 版本，这里是 v1。

- `kind: RoleBinding`：指定 Kubernetes 对象类型为 RoleBinding。

- `metadata:`：定义 Kubernetes 对象的元数据，包括名称和命名空间。

- `name: deployment-restart`：指定创建的 RoleBinding 的名称为 deployment-restart。

- `namespace: default`：指定命名空间为 default。

- `roleRef:`：引用要绑定的角色。

- `apiGroup: rbac.authorization.k8s.io`：指定 RBAC API 组。

- `kind: Role`：指定角色类型为 Role。

- `name: deployment-restart`：指定要绑定的角色的名称为 deployment-restart。

- `subjects:`：指定要绑定角色的主体（例如 ServiceAccount）。

- `- kind: ServiceAccount`：指定主体对象的类型为 ServiceAccount。

- `name: deployment-restart`：指定主体对象的名称为 deployment-restart。

- `namespace: default`：指定主体对象所在的命名空间为 default。

### CronJob

```
---
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: deployment-restart
  namespace: default
spec:
  concurrencyPolicy: Forbid
  # cron spec of time, here, 8 o'clock
  schedule: "0 1 * * *"
  jobTemplate:
    spec:
      backoffLimit:
        # this has very low chance of failing, as all this does
        # is prompt kubernetes to schedule new replica set for
        # the deployment
        2
      activeDeadlineSeconds:
        # timeout, makes most sense with "waiting for rollout" variant specified below
        600
      template:
        spec:
          serviceAccountName:
            # name of the service account configured above
            deployment-restart
          restartPolicy: Never
          containers:
            - name: kubectl
              image:
                # probably any kubectl image will do,
                # optionaly specify version, but this
                # should not be necessary, as long the
                # version of kubectl is new enough to
                # have `rollout restart`
                bitnami/kubectl
              command:
                - "kubectl"
                - "rollout"
                - "restart"
                - "deployment/my-fast-and-robust-service"
```

- `---`：指示 Kubernetes YAML 文件的开始。

- `apiVersion: batch/v1beta1`：指定批处理 API 版本，这里是 v1beta1。

- `kind: CronJob`：指定 Kubernetes 对象类型为 CronJob。

- `metadata:`：定义 Kubernetes 对象的元数据，包括名称和命名空间。

- `name: deployment-restart`：指定创建的 `CronJob` 的名称为 deployment-restart。

- `namespace: default`：指定命名空间为 `default`。

- `spec:`：定义 CronJob 的规范。

- `concurrencyPolicy: Forbid`：指定并发策略为 Forbid，即如果上一个任务还未完成，则不会启动新的任务。

- `schedule: "0 1 * * *"`：指定 CronJob 的运行频率，这里是每天的凌晨 1 点。

- `jobTemplate:`：定义要执行的作业模板。

- `spec:`：指定作业的规范。

- `backoffLimit:`：定义作业的退避限制，即在失败后重试此作业的次数。这里设置为 2。

- `activeDeadlineSeconds:`：定义作业的运行时间截止日期（以秒为单位）。这里设置为 600 秒。

- `template:`：定义作业的 Pod 模板。

- `spec:`：指定 Pod 的规范。

- `serviceAccountName:`：指定 Pod 使用的 ServiceAccount 的名称，这里是 deployment-restart。

- `restartPolicy: Never`：定义 Pod 的重启策略为 Never，即当 Pod 终止时不会自动重启。

- `containers:`：定义 Pod 中的容器列表。

- `- name: kubectl`：指定容器的名称为 kubectl。

- `image: bitnami/kubectl`：指定使用的 kubectl 容器镜像。

- `command:`：指定要在容器中执行的命令列表。

- `- "kubectl"`：指定要执行的第一个命令为 kubectl。

- `- "rollout"`：指定要执行的第二个命令为 rollout。

- `- "restart"`：指定要执行的第三个命令为 restart。

- `- "deployment/my-fast-and-robust-service"`：指定要重启的 Deployment 的名称为 `my-fast-and-robust-service`。

总之，这个 `YAML` 配置文件定义了 `CronJob`，并使用 kubectl 命令重启指定的 Deployment。该 CronJob 将定期运行，并确保 Deployment 在运行过程中保持稳定性和可用性。
