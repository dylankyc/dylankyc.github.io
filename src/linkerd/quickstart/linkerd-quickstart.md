# Hands on Linkerd

- [1、安装](#1%E5%AE%89%E8%A3%85)
- [2、安装 linkerd](#2%E5%AE%89%E8%A3%85-linkerd)
- [3、下载镜像](#3%E4%B8%8B%E8%BD%BD%E9%95%9C%E5%83%8F)
- [4、开启 dashboard](#4%E5%BC%80%E5%90%AF-dashboard)
- [5、安装示例工程 emojivoto](#5%E5%AE%89%E8%A3%85%E7%A4%BA%E4%BE%8B%E5%B7%A5%E7%A8%8B-emojivoto)
- [6、注入 linkerd 到 emojivoto](#6%E6%B3%A8%E5%85%A5-linkerd-%E5%88%B0-emojivoto)
- [7、通过 grafana 查看监控](#7%E9%80%9A%E8%BF%87-grafana-%E6%9F%A5%E7%9C%8B%E7%9B%91%E6%8E%A7)
- [8、调试](#8%E8%B0%83%E8%AF%95)
- [Refs](#refs)

# 1、安装

首先搭建 kubernetes 环境，我用的是 Docker Desktop for Mac：

# 2、安装 linkerd

curl 安装，并将 `linkerd` 加到 PATH 路径:

```bash
curl -sL https://run.linkerd.io/install | sh
export PATH=$PATH:$HOME/.linkerd2/bin
```

检查下版本（这里是 `statble-2.3.2`）：

```bash
$ linkerd version
Client version: stable-2.3.2
Server version: stable-2.3.2
```

# 3、下载镜像

众所周知的原因 gci.io 无法访问，需要另辟蹊径替换镜像地址：

```bash
linkerd install | \
    sed -e 's/gcr.io\/linkerd-io\//dylankyc\/gcr.io_linkerd-io_/g' \
    kubectl apply -f -
```

这里是输出：

```
namespace/linkerd created
configmap/linkerd-config created
serviceaccount/linkerd-identity created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-identity unchanged
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-identity unchanged
service/linkerd-identity created
secret/linkerd-identity-issuer created
deployment.extensions/linkerd-identity created
serviceaccount/linkerd-controller created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-controller unchanged
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-controller unchanged
service/linkerd-controller-api created
service/linkerd-destination created
deployment.extensions/linkerd-controller created
customresourcedefinition.apiextensions.k8s.io/serviceprofiles.linkerd.io unchanged
serviceaccount/linkerd-web created
service/linkerd-web created
deployment.extensions/linkerd-web created
serviceaccount/linkerd-prometheus created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-prometheus unchanged
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-prometheus unchanged
service/linkerd-prometheus created
deployment.extensions/linkerd-prometheus created
configmap/linkerd-prometheus-config created
serviceaccount/linkerd-grafana created
service/linkerd-grafana created
deployment.extensions/linkerd-grafana created
configmap/linkerd-grafana-config created
serviceaccount/linkerd-sp-validator created
clusterrole.rbac.authorization.k8s.io/linkerd-linkerd-sp-validator unchanged
clusterrolebinding.rbac.authorization.k8s.io/linkerd-linkerd-sp-validator configured
service/linkerd-sp-validator created
deployment.extensions/linkerd-sp-validator created
```

或者你也先将 `linkerd install` 输出的 yaml 保存起来：

```bash
linkerd install > linkerd.yaml
# replace gcr.io image with docker hub; for linux, replace `sed -i ""` with  `sed -i`
sed -i "" -e 's/gcr.io\/linkerd-io\//dylankyc\/gcr.io_linkerd-io_/g' linkerd.yaml
```

再通过 kubectl 安装：

```bash
kubectl apply -f linkerd.yaml
```

喝杯咖啡之后 linkerd 就安装完成了，如果不行就两杯。

# 4、开启 dashboard

```bash
linkerd dashboard &
```

默认会随机分配一个端口如 50750，打开 http://127.0.0.1:50750/overview，通过 dashboard 可以探索 linkerd mesh 内部组件。

# 5、安装示例工程 emojivoto

```bash
curl -sL https://run.linkerd.io/emojivoto.yml | kubectl apply -f -
```

将容器 80 端口转发到本地 8080，这样通过 localhost:8080 就能访问 emojivoto 服务了。

```bash
kubectl -n emojivoto port-forward svc/web-svc 8080:80 &
```

由于通过 kubectl 安装，此时没有注入 linkerd sidecar，所以 Meshed 显示 0/1。

# 6、注入 linkerd 到 emojivoto

```bash
kubectl get -n emojivoto deploy -o yaml \
| linkerd inject - \
| kubectl apply -f -
```

注入 linkerd 的 deployment 在 dashboard 上可以查看到显示为 `Meshed`。

注入 linkderd 需要操纵的是 deployment，如：

```bash
linkerd inject deployment.yml \
| kubectl apply -f -
```

它会把 linkerd 作为 sidecar inject 到 pod 里，并配置 iptables。

```bash
$ kubectl get -n emojivoto deploy -o yaml | linkerd inject - | kubectl apply -f -

deployment "emoji" injected
deployment "vote-bot" injected
deployment "voting" injected
deployment "web" injected

deployment.extensions/emoji configured
deployment.extensions/vote-bot configured
deployment.extensions/voting configured
deployment.extensions/web configured
```

# 7、通过 grafana 查看监控

访问 `emojivoto` 几次，可以在 grafana 面板 http://127.0.0.1:50750/grafana 查看到服务的历史数据，如 Success rate, Request rate, Latency distribution percentiles 等。

# 8、调试

为了演示，demo 故意在代码里埋了一些错误。

点开 `deployment/web`，可以看到 `deploy/web` 接收 `deploy/vote-bot` 请求，同时也会给 `deploy/emoji` 和 `deploy/voting` 发送请求。

TODO:

但是值得注意的是，`deploy/vote-bot` 和 `deploy/voting` 的成功率都不是 100%，由于 `deploy/vote-bot` 调用 `deploy/web` 进而调用 `deploy/voting`，
还可以看出，对 web 而言，`deploy/vote-bot` (对应 PATH `/api/vote`）是入口，而 `deploy/voting` （对应 PATH `/emojivoto.v1.VotingService/VoteDoughnut`）是出口，我们可以猜想错误因 `deploy/voting` 而起。

我们还可以点击 tap，通过查看只针对这一接口 `/emojivoto.v1.VotingService/VoteDoughnut` 的请求来进一步定位错误。

通过图中 Unknown 的消息及 grpc 针对 Unknown code 的说明文档（https://godoc.org/google.golang.org/grpc/codes#Code），可知这个接口有异常。

我们还可以通过查看代码来验证：https://github.com/BuoyantIO/emojivoto/blob/master/emojivoto-voting-svc/api/api.go#L22，我们可以看到 doughnut（甜甜圈）那里报错了，从而定位坑。

# Refs

https://linkerd.io/2/tasks/debugging-your-service/
