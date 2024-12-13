# Pod security to access aws s3 files

- [Intro](#intro)
- [创建 `IAM policy`](#%E5%88%9B%E5%BB%BA-iam-policy)
- [创建 service account 并和 `IAM policy` 绑定](#%E5%88%9B%E5%BB%BA-service-account-%E5%B9%B6%E5%92%8C-iam-policy-%E7%BB%91%E5%AE%9A)
- [修改 Deployment 的 spec.template.spec.serviceAccountName](#%E4%BF%AE%E6%94%B9-deployment-%E7%9A%84-spectemplatespecserviceaccountname)
- [Refs](#refs)

# Intro

部署在 `kubernetes` 的应用的安全性是值得考量的，为了提高安全性，我们可以通过以下的方式来保证应用的安全性，比如：

- 给 `kubernetes` 的 worker node 添加 `IAM policy`，这样 worker node 上运行的 pod 就可以通过 worker node 的 `IAM policy` 的权限来访问应用的资源
- 给 `kubernetes` 的 service account 绑定 `IAM policy`，这样 service account 可以通过 `IAM policy` 的权限来访问应用的资源

当然，worker node 添加 `IAM policy` 这种方式粒度比较粗糙，不太推荐，我们可以通过更精细的方式来保证应用的安全性。

# 创建 `IAM policy`

以下的脚本会创建一个 `IAM policy`，这个 `IAM policy` 可以被 `kubernetes` 的 service account 绑定，以便 service account 可以通过 `IAM policy` 的权限来访问应用的资源。

```bash
#!/bin/bash
POLICY_NAME="my-very-sensitive-bucket-service-account-policy"
K8S_SERVICE_ACCOUNT_ACCESS_S3_BUCKET_POLICY_FILE="k8s-service-account-access-s3-bucket-policy.json"

cat > $K8S_SERVICE_ACCOUNT_ACCESS_S3_BUCKET_POLICY_FILE <<EOF
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Sid": "statement1",
         "Effect": "Allow",
         "Action": [
            "s3:ListBucket",
            "s3:GetObject"
         ],
         "Resource": [
            "arn:aws:s3:::test-bucket/my-very-sensitive-data/*"
         ]
      }
   ]
}
EOF

aws iam create-policy \
    --policy-name $POLICY_NAME \
    --policy-document file://$K8S_SERVICE_ACCOUNT_ACCESS_S3_BUCKET_POLICY_FILE
```

上面的脚本首先会创建一个 `IAM policy`，这个 policy 会授予对 `S3` 的某个 bucket 的权限: `ListBucket` 和 `GetObject`。

# 创建 service account 并和 `IAM policy` 绑定

有了 `IAM policy`，我们可以创建一个 service account，并将 `IAM policy` 绑定到 service account 上。

```bash
account_id="12345678999"
cluster="test"
namespace="financial"
name="my-very-sensitive-bucket-service-account"
eksctl create iamserviceaccount \
  --name $name \
  --namespace $namespace \
  --cluster $cluster \
  --attach-policy-arn arn:aws:iam::12345678999:policy/$POLICY_NAME \
  --approve
```

上面的脚本会通过 `eksctl` 创建一个名为 `my-very-sensitive-bucket-service-account` 的 `kubernetes` 的 service account，并且将上面创建的 policy 关联到这个 service account 上。

我们可以通过 `kubectl` 命令来验证这个 service account 的创建成功: `kubectl -n financial get sa my-very-sensitive-bucket-service-account -oyaml`

输出:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::12345678999:role/eksctl-test-addon-iamserviceaccount-financial-Role1-3F4F5F6F7F8F
  creationTimestamp: "2022-04-18T06:39:41Z"
  labels:
    app.kubernetes.io/managed-by: eksctl
  name: my-very-sensitive-bucket-service-account
  namespace: financial
  resourceVersion: "123123123"
  uid: 13232333-ffee-ffee-ffee-39393929339
secrets:
  - name: my-very-sensitive-bucket-service-account-token-ffee33
```

# 修改 Deployment 的 spec.template.spec.serviceAccountName

有了 service account，我们可以修改 `Deployment` 的 `spec.template.spec.serviceAccountName`，这样 `Deployment` 就会使用这个 service account 了。

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
  namespace: logging
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
      serviceAccountName: my-very-sensitive-bucket-service-account #this is where your service account is specified
      hostNetwork: true
```

# Refs

https://kubernetes-on-aws.readthedocs.io/en/latest/user-guide/service-accounts.html

assign iam role to service account(eksctl did it)
https://docs.amazonaws.cn/en_us/eks/latest/userguide/specify-service-account-role.html

https://mr3docs.datamonad.com/docs/k8s/eks/access-s3/

https://aws.amazon.com/cn/premiumsupport/knowledge-center/eks-restrict-s3-bucket/

https://dzone.com/articles/how-to-use-aws-iam-role-on-aws-eks-pods
