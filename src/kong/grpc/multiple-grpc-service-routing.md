# Multiple GRPC Service Routing in Kong

<!-- toc -->

# Into

这里演示了利用 kong 对多个 grpc 服务进行路由的方法。

# 1.在 kubernetes 安装 kong

```shell
kubecstl apply -f https://bit.ly/kong-ingress-dbless
```

# 2.安装完查看 kong 服务情况

```shell
$ kubectl -n kong get all
NAME                                READY   STATUS    RESTARTS   AGE
pod/ingress-kong-1122334455-mzfnx   2/2     Running   0          23h

NAME                              TYPE           CLUSTER-IP      EXTERNAL-IP                                                                             PORT(S)                      AGE
service/kong-proxy                LoadBalancer   172.20.179.34   a111222333444555666777888999000f-1122334455667788.elb.cn-northwest-1.amazonaws.com.cn   80:30004/TCP,443:32418/TCP   23h
service/kong-validation-webhook   ClusterIP      172.20.59.181   <none>                                                                                  443/TCP                      23h

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-kong   1/1     1            1           23h

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-kong-1122334455   1         1         1       23h
```

将 AWS ELB 保存在变量 `ELB` 中

```shell
ELB=$(kubectl -n kong get svc -o=jsonpath="{.items[0].status.loadBalancer.ingress[0].hostname}")

echo $ELB
# a111222333444555666777888999000f-1122334455667788.elb.cn-northwest-1.amazonaws.com.cn
```

# 3.安装 grpcbin 服务

```shell
kubectl apply -f https://bit.ly/grpcbin-service
```

给 service 打 patch，让 kong 使用 gRPC 协议来和 upstream 通信，grpcbin 9001 服务需要指定 protocol 为 `grpcs`

```shell
kubectl patch svc grpcbin -p '{"metadata":{"annotations":{"konghq.com/protocol":"grpcs"}}}'
```

安装 ingress

    kubectl apply -f ingress.yaml

`ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: grpcbin-grpc
  annotations:
    konghq.com/protocols: grpc,grpcs
spec:
  rules:
    - http:
        paths:
          - path: /hello.HelloService
            backend:
              serviceName: grpcbin
              servicePort: 9001
```

下面进行测试。

测试前需要准备 protobuf 文件

`hello.proto`

```protobuf
// based on https://grpc.io/docs/guides/concepts.html

syntax = "proto2";

package hello;

service HelloService {
  rpc SayHello(HelloRequest) returns (HelloResponse);
  rpc LotsOfReplies(HelloRequest) returns (stream HelloResponse);
  rpc LotsOfGreetings(stream HelloRequest) returns (HelloResponse);
  rpc BidiHello(stream HelloRequest) returns (stream HelloResponse);
}

message HelloRequest {
  optional string greeting = 1;
}

message HelloResponse {
  required string reply = 1;
}
```

通过 grpcurl 来测试

```shell
$ grpcurl -v -d '{"greeting": "Kong Hello world!"}' -proto hello.proto -insecure $ELB:443 hello.HelloService.SayHello

Resolved method descriptor:
rpc SayHello ( .hello.HelloRequest ) returns ( .hello.HelloResponse );

Request metadata to send:
(empty)

Response headers received:
content-type: application/grpc
date: Wed, 13 May 2020 02:29:43 GMT
server: openresty
trailer: Grpc-Status
trailer: Grpc-Message
trailer: Grpc-Status-Details-Bin
via: kong/2.0.4
x-kong-proxy-latency: 1
x-kong-upstream-latency: 13

Response contents:
{
  "reply": "hello Kong Hello world!"
}

Response trailers received:
(empty)
Sent 1 request and received 1 response
```

# 4.安装 helloworld-grpc 服务

```shell
kubectl apply -f helloworld-grpc.yaml
```

`helloworld-grpc.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld
spec:
  selector:
    matchLabels:
      app: helloworld
  replicas: 1
  template:
    metadata:
      labels:
        app: helloworld
    spec:
      containers:
        - name: helloworld
          image: "quay.io/dylankyc/grpc-examples-helloworld-server"
          resources:
            limits:
              cpu: 64m
              memory: 128Mi
            requests:
              cpu: 10m
              memory: 64Mi
---
apiVersion: v1
kind: Service
metadata:
  name: helloworld
  labels:
    app: helloworld
  annotations:
    konghq.com/protocol: grpc
spec:
  # type: ClusterIP
  selector:
    app: helloworld
  ports:
    - port: 50051
      targetPort: 50051
      name: grpc
      # protocol: TCP
```

给 service 打 patch，让 kong 使用 gRPC 协议来和 upstream 通信，因为 helloworld 采用非 TLS 方式，这里指定 protocol 为 `grpc`

```bash
kubectl patch svc helloworld -p '{"metadata":{"annotations":{"konghq.com/protocol":"grpc"}}}'
```

安装 ingresss

```bash
kubectl apply -f ingress.yaml
```

`ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: helloworld-grpc
  annotations:
    konghq.com/protocols: grpc,grpcs
spec:
  rules:
    - http:
        paths:
          - path: /helloworld.Greeter
            backend:
              serviceName: helloworld
              servicePort: 50051
```

下面进行测试。

测试前需要准备 protobuf 文件

`helloworld.proto`

```protobuf
// protoc -I helloworld/ helloworld/helloworld.proto --go_out=plugins=grpc:helloworld

syntax = "proto3";

// option go_package = "github.com/dylankyc/grpc-examples/helloworld/internal/pb/helloworld";
// option go_package = "../pb/helloworld;helloworld";
// OK
option go_package = "pb/helloworld;helloworld";

// NOT OK
// option go_package = "helloworld";

package helloworld;

// The greeter service definition.
service Greeter {
    // Sends a greeting
    rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```

通过 grpcurl 来测试

```shell
$ grpcurl -v -d '{"name": "Kong Hello world!"}' -insecure -proto helloworld.proto $ELB:443 helloworld.Greeter.SayHello

Resolved method descriptor:
// Sends a greeting
rpc SayHello ( .helloworld.HelloRequest ) returns ( .helloworld.HelloReply );

Request metadata to send:
(empty)

Response headers received:
content-type: application/grpc
date: Wed, 13 May 2020 02:55:01 GMT
server: openresty
via: kong/2.0.4
x-kong-proxy-latency: 0
x-kong-upstream-latency: 2

Response contents:
{
  "message": "Hello Kong Hello world!"
}

Response trailers received:
(empty)
Sent 1 request and received 1 response
```
