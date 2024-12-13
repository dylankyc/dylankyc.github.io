# Kubebuilder quickstart

- [Intro](#intro)
- [kubebuilder](#kubebuilder)
  - [首先安装 kubebuilder](#%E9%A6%96%E5%85%88%E5%AE%89%E8%A3%85-kubebuilder)
  - [其次新建 kubebuilder 工程](#%E5%85%B6%E6%AC%A1%E6%96%B0%E5%BB%BA-kubebuilder-%E5%B7%A5%E7%A8%8B)
  - [新建 group version kind](#%E6%96%B0%E5%BB%BA-group-version-kind)
  - [定义 CRD Spec](#%E5%AE%9A%E4%B9%89-crd-spec)
  - [实现自定义 CRD 的 Reconcile](#%E5%AE%9E%E7%8E%B0%E8%87%AA%E5%AE%9A%E4%B9%89-crd-%E7%9A%84-reconcile)
  - [Reconcile 内部实现](#reconcile-%E5%86%85%E9%83%A8%E5%AE%9E%E7%8E%B0)
    - [Reconcile logic](#reconcile-logic)
    - [处理 CRD](#%E5%A4%84%E7%90%86-crd)
    - [finalizer](#finalizer)
    - [deployment](#deployment)
    - [service](#service)
    - [ingress](#ingress)
- [测试](#%E6%B5%8B%E8%AF%95)

# Intro

我们知道 k8s 部署应用非常繁琐，首先要配置 `deployment`， `service`，外部流量的引入还要使用 `ingress`，这就要同时维护三个文件，而利用 kubernetes operator 能大大简化部署操作。

我们采用 `kuberbuilder` 这个 `framework` 来开发 kubernenetes operator，要完成**一个配置文件完成「kubectl apply 新建三连」的功能**。

# kubebuilder

## 首先安装 kubebuilder

```shell
os=$(go env GOOS)
arch=$(go env GOARCH)
v=2.3.1
# download kubebuilder and extract it to tmp
curl -L https://go.kubebuilder.io/dl/${v}/${os}/${arch} | tar -xz -C /tmp/

# move to a long-term location and put it on your path
# (you'll need to set the KUBEBUILDER_ASSETS env var if you put it somewhere else)
sudo mv /tmp/kubebuilder_${v}_${os}_${arch} /usr/local/kubebuilder
export PATH=$PATH:/usr/local/kubebuilder/bin
```

## 其次新建 kubebuilder 工程

```shell
kubebuilder init --domain example.com CustomImageDeploy
```

## 新建 group version kind

```shell
kubebuilder create api --group customimagedeploy --version v1 --kind CustomImageDeploy
```

## 定义 CRD Spec

接下来我们需要定义 CRD 的 Spec

```go
// CustomImageDeploySpec defines the desired state of CustomImageDeploy
type CustomImageDeploySpec struct {
    // INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
    // Important: Run "make" to regenerate code after modifying this file

    // Image is the docker image with version info of CustomImageDeploy.
    Image string `json:"image,omitempty"`

    // Size is the number of pods to run
    Size int32 `json:"size"`

    // Port is the port of container
    Port int32 `json:"port"`
}
```

这里定义了要运行的 docker image(`Image`)，数量(`Size`)和 container 的端口(`Port`)

## 实现自定义 CRD 的 Reconcile

```go
func (r *CustomImageDeployReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
}
```

Reconcile 函数实现了 operator 的功能。

## Reconcile 内部实现

### Reconcile logic

- 获取 CRD，处理 `client.IgnoreNotFound(err)`，此时表示 CRD 被删除，Reconcile 返回 `ctrl.Result{}, nil` ，Reconcile loop 结束

- 处理 `finalizer`，如果`ObjectMeta.DeletionTimestamp.IsZero()` 则表示未正在被删除，我们需要给 CRD 的 `ObjectMeta` 添加 `finalizer`；否则我们判断 CRD 的 ObjectMeta 是否包含 finalizer，并删除其他外部资源，删除成功之后清除 ObjectMeta 中的 `finalizer`，剩下删除的工作交给 `kubernetes` 去处理

- 获取 `ingress`，处理 `client.IgnoreNotFound(err)`，此时表示 `ingress` 尚未被创建，则需要调用 `r.Client.Create` 来创建

- 获取 `deployment`，处理 `client.IgnoreNotFound(err)`，此时表示 `deployment` 尚未被创建，则需要调用 `r.Client.Create` 来创建

- 获取 `service`，处理 `client.IgnoreNotFound(err)`，此时表示 `service` 尚未被创建，则需要调用 `r.Client.Create` 来创建

- 其他外部资源的处理，由于我们未使用其他外部资源，这里忽略

### 处理 CRD

```go
log := r.Log.WithValues("customimagedeploy", req.NamespacedName)

log.Info("[CustomImageDeployReconciler::Reconsile]", "req: ", req)

cid := &customimagedeployv1.CustomImageDeploy{}
err := r.Client.Get(context.TODO(), req.NamespacedName, cid)
log.Info("Begin to use finalizer", "cid : ", cid)

if err != nil {
    //if errors.IsNotFound(err) {
    //    // Request object not found, could have been deleted after reconcile req.
    //    // Owned objects are automatically garbage collected. For additional cleanup logic use finalizers.
    //    // Return and don't requeue
    //    log.Info("CustomImageDeploy resource not found. Ignoring since object must be deleted.")
    //    return ctrl.Result{}, nil
    //}
    //return ctrl.Result{}, err
    log.Info("[CustomImageDeployReconciler::Reconsile] get err != nil", "err: ", err)
    return ctrl.Result{}, client.IgnoreNotFound(err)
}
```

### finalizer

```go
f := "customimagedeploy.finalizers.example.com"
if cid.ObjectMeta.DeletionTimestamp.IsZero() {
    log.Info("DeletionTimestamp.IsZero")

    // The object is not being deleted, so if it does not have our finalizer,
    // then lets add the finalizer and update the object.
    if !containsString(cid.ObjectMeta.Finalizers, f) {
        cid.ObjectMeta.Finalizers = append(cid.ObjectMeta.Finalizers, f)
        if err := r.Update(context.Background(), cid); err != nil {
            return reconcile.Result{}, err
        }
    }
} else {
    // The object is being deleted
    if containsString(cid.ObjectMeta.Finalizers, f) {
        // our finalizer is present, so lets handle our external dependency
        if err := r.deleteExternalDependency(cid); err != nil {
            // if fail to delete the external dependency here, return with error
            // so that it can be retried
            return reconcile.Result{}, err
        }

        // remove our finalizer from the list and update it.
        cid.ObjectMeta.Finalizers = removeString(cid.ObjectMeta.Finalizers, f)
        if err := r.Update(context.Background(), cid); err != nil {
            return reconcile.Result{}, err
        }
    }

    // Our finalizer has finished, so the reconciler can do nothing.
    return reconcile.Result{}, nil
}
```

其中 `containsString` 方法判断 `finalizer` 数组里是否包含我们预先定义的 `finalizer`

```go
func containsString(slice []string, s string) bool {
    for _, item := range slice {
        if item == s {
            return true
        }
    }
    return false
}
```

### deployment

```go
// check if Deployment already exists, if not create a new one
deployment := &appsv1.Deployment{}
log.Info("Getting the deployment.", "cid: ", cid)
err = r.Client.Get(context.Background(), types.NamespacedName{Name: cid.Name, Namespace: cid.Namespace}, deployment)
if errors.IsNotFound(err) {
    dep := r.deploymentForCustomImageDeploy(cid)
    log.Info("Creating a new deployment.", "Namespace: ", dep.Namespace, "Name: ", dep.Name)
    err = r.Client.Create(context.Background(), dep)
    if err != nil {
        log.Error(err, "Failed to create a new deployment", "Namespace: ", dep.Namespace, "Name: ", dep.Name)
        return ctrl.Result{}, err
    }
}
if err != nil {
    log.Error(err, "Failed to create a new deployment")
    return ctrl.Result{}, err
}

// ensure the size
size := cid.Spec.Size
if deployment.Spec.Replicas == nil {
    // replicas is nil, requeue
    log.Info("deployment.Spec.Replicas is nil")
    return ctrl.Result{RequeueAfter: time.Second * 5}, nil
}

if *deployment.Spec.Replicas != size {
    deployment.Spec.Replicas = &size
    err = r.Client.Update(context.Background(), deployment)
    if err != nil {
        log.Error(err, "Failed to udpate deployment", "Namespace: ", deployment.Namespace, "Name: ", deployment.Name)
        return ctrl.Result{}, err
    }
    // size not match, requeue
    return ctrl.Result{RequeueAfter: time.Second * 5}, nil
}
```

其中 `deploymentForCustomImageDeploy` 会准备需要创建 `deployment` 的 Spec

```go
// deploymentForMemcached returns a Deployment object
func (r *CustomImageDeployReconciler) deploymentForCustomImageDeploy(c *customimagedeployv1.CustomImageDeploy) *appsv1.Deployment {
    replicas := c.Spec.Size
    image := c.Spec.Image
    name := c.Name
    port := c.Spec.Port

    ls := labelsForCustomImageDeploy(name)

    dep := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      c.Name,
            Namespace: c.Namespace,
            Labels:    ls,
        },
        Spec: appsv1.DeploymentSpec{
            Replicas: &replicas,
            Selector: &metav1.LabelSelector{
                MatchLabels: ls,
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: ls,
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{{
                        Image: image,
                        Name:  name,
                        Ports: []corev1.ContainerPort{{
                            ContainerPort: port,
                            // Name:          name, // Name is optinal, no more than 15 characters
                        }},
                    }},
                },
            },
        },
    }

    log := r.Log.WithValues("CustomImageDeployReconciler", "deploymentForCustomImageDeploy")

    // Set Memcached instance as the owner of the Deployment.
    if err := ctrl.SetControllerReference(c, dep, r.Scheme); err != nil {
        log.Info("SetControllerReference", "error : ", err)
    } //todo check how to get the schema

    return dep
}
```

其中辅助函数 `labelsForCustomImageDeploy` 用于生成 labels

```go
// labelsForCustomImageDeploy returns the labels for selecting the resources
// belonging to the given custom-image-deploy CR name.
func labelsForCustomImageDeploy(name string) map[string]string {
    return map[string]string{"app": name, "managed_by": "custom-image-deploy"}
}
```

### service

```go
// check if Service already exists, if not create a new one
service := &corev1.Service{}
log.Info("Getting the service.", "cid: ", cid)
err = r.Client.Get(context.Background(), types.NamespacedName{Name: cid.Name, Namespace: cid.Namespace}, service)
if errors.IsNotFound(err) {
    svc := r.serviceForCustomImageDeploy(cid)
    log.Info("Creating a new service.", "Namespace: ", svc.Namespace, "Name: ", svc.Name)
    err = r.Client.Create(context.Background(), svc)
    if err != nil {
        log.Error(err, "Failed to create a new service", "Namespace: ", svc.Namespace, "Name: ", svc.Name)
        return ctrl.Result{}, err
    }
}
if err != nil {
    log.Error(err, "Failed to create a new service")
    return ctrl.Result{}, err
}

// make sure service is created(has a clusterip)
if service.Spec.ClusterIP == "" {
    return ctrl.Result{RequeueAfter: time.Second * 5}, nil
}
```

### ingress

```go
// check if Ingress already exists, if not create a new one
ing := &networking.Ingress{}
err = r.Client.Get(context.TODO(), types.NamespacedName{Name: cid.Name, Namespace: cid.Namespace}, ing)
if errors.IsNotFound(err) {
    log.Info("Creating a new ingress.", "cid: ", cid)
    ing := r.ingressForCustomImageDeploy(cid)
    log.Info("Creating a new ingress.", "Namespace: ", ing.Namespace, "Name: ", ing.Name)
    err = r.Client.Create(context.TODO(), ing)
    if err != nil {
        log.Error(err, "Failed to create a new ingress", "Namespace: ", ing.Namespace, "Name: ", ing.Name)
        return ctrl.Result{}, err
    }
}

if err != nil {
    log.Error(err, "Failed to create a new ingress", "Namespace: ", ing.Namespace, "Name: ", ing.Name)
    return ctrl.Result{}, err
}

if len(ing.Status.LoadBalancer.Ingress) == 0 {
    return ctrl.Result{RequeueAfter: time.Second * 5}, nil
}
```

# 测试

以 `nginx` 为例，我们需要创建 nginx deployment、nginx service 和 nginx ingress。以往我们都会准备三个文件：`deployment.yaml,` `service.yaml`, `ingress.yaml`，现在我们只需一个文件就可以了 `nginx.yaml`。

```yaml
apiVersion: customimagedeploy.example.com/v1
kind: CustomImageDeploy
metadata:
  name: customimagedeploy-nginx
spec:
  # Add fields here
  size: 1
  port: 80
  image: "nginx:1.17"
```

一键部署：

```shell
kubectl apply -f nginx.yaml
```

查看结果：

```shell
$ k get pod,svc,deploy,rs,ing -l managed_by=custom-image-deploy
NAME                                           READY   STATUS    RESTARTS   AGE
pod/customimagedeploy-nginx-7f55f7c585-pb9bm   1/1     Running   0          6m20s

NAME                              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/customimagedeploy-nginx   ClusterIP   172.20.50.80   <none>        80/TCP    6m20s

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/customimagedeploy-nginx   1/1     1            1           6m20s

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/customimagedeploy-nginx-7f55f7c585   1         1         1       6m20s

NAME                                         HOSTS                             ADDRESS                                                                                 PORTS   AGE
ingress.extensions/customimagedeploy-nginx   customimagedeploy-nginx.default   a123456789012345623242424424-1314151151515.elb.cn-northwest-1.amazonaws.com.cn   80      84s
```

由于我们部署了 kong 作为 api gateway，我们可以通过访问 load balancer 地址来测试一下是否能正确访问刚刚部署的 nginx 服务

```shell
curl -H "Host: customimagedeploy-nginx.default" \
a123456789012345623242424424-1314151151515.elb.cn-northwest-1.amazonaws.com.cn
```

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Welcome to nginx!</title>
    <style>
      body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
      }
    </style>
  </head>
  <body>
    <h1>Welcome to nginx!</h1>
    <p>
      If you see this page, the nginx web server is successfully installed and
      working. Further configuration is required.
    </p>

    <p>
      For online documentation and support please refer to
      <a href="http://nginx.org/">nginx.org</a>.<br />
      Commercial support is available at
      <a href="http://nginx.com/">nginx.com</a>.
    </p>

    <p><em>Thank you for using nginx.</em></p>
  </body>
</html>
```
