# Extend Disk in ClickHouse Cluster

<!-- toc -->

# Disk Extend Progress in ClickHouse Operator

_How to Extend Disk Size in a ClickHouse Cluster and Monitor Its Progress_

Managing disk resources is a crucial part of maintaining performance as your ClickHouse cluster grows over time. In this post, we'll explore how to extend the disk size when using the ClickHouse Operator. We'll walk through the configuration changes, review operator logs during the process, and even examine Kubernetes events that show how pods are recreated as part of the scaling routine.

---

## Introduction

A common challenge when operating a ClickHouse cluster is ensuring that disk storage grows along with your data needs. In our deployment—with two shards and two replicas—we encountered a situation where the previous 800Gi disk was no longer sufficient. Fortunately, the ClickHouse Operator makes scaling simple by allowing a configuration update.

In this post, we'll explain the step-by-step process for extending disk size, from updating the `cluster.yaml` to monitoring the results via operator logs and Kubernetes events.

---

## Background: Our ClickHouse Cluster Configuration

For context, here is a simplified version of our ClickHouse cluster configuration. In our deployment, the ClickHouse Operator uses a template that defines various resources, such as the data volume, pod, and service configurations:

```yaml:src/clickhouse-operator/disk-extend-progress.blog.md
apiVersion: "clickhouse.altinity.com/v1"
kind: "ClickHouseInstallation"
metadata:
  name: "logs"
spec:
  defaults:
    templates:
      dataVolumeClaimTemplate: data-volumeclaim-template
      podTemplate: clickhouse:23.3
      serviceTemplate: svc-template
  configuration:
    users:
      # default user should be security to localhost and interhost connections
      # operator creates host_regxp expression for that
      # default/networks/host_regexp: \.chi-test-011-secure-user-[^.]+-\d+-\d+\.test.svc.cluster.local$
      default/profile: default
      default/quota: default
      default/networks/ip:
        - 127.0.0.1
        - 127.0.0.2
      # user1 with a password
      user1/password: topsecret
      user1/networks/ip: "::/0"
    zookeeper:
      nodes:
        - host: zookeeper-0.zookeepers.zoo3ns
        - host: zookeeper-1.zookeepers.zoo3ns
        - host: zookeeper-2.zookeepers.zoo3ns
    clusters:
      - name: logs
        layout:
          shardsCount: 2
          replicasCount: 2

  templates:
    volumeClaimTemplates:
      - name: data-volumeclaim-template
        # NOTE: previous name: default
        #- name: default
        # NOTE: aws ebs volume not deleted after the cluster is deleted
        reclaimPolicy: Retain
        spec:
          storageClassName: ebs-sc-retain
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 800Gi
    serviceTemplates:
      - name: svc-template
        spec:
          ports:
            - name: http
              port: 8123
            - name: tcp
              port: 9000
          type: ClusterIP
    podTemplates:
      - name: clickhouse:23.3
        # NOTE: ERROR using 👇
        #        zone:
        #          values:
        #            - "clickhouse"
        spec:
          nodeSelector:
            intend: monitoring
            kubernetes.io/arch: arm64
          containers:
            - name: clickhouse-pod
              image: 321321321321.dkr.ecr.cn-northwest-1.amazonaws.com.cn/clickhouse/clickhouse-server:23.3
```

_Note:_ In the above configuration, the primary storage request is set to **800Gi**.

---

## Updating the Cluster Configuration

The disk extension process begins by updating the storage allocation in the cluster configuration. To extend the disk from 800Gi to 1200Gi, you simply change the storage request in the configuration file.

Below is an example diff that shows this change:

```diff:src/clickhouse-operator/disk-extend-progress.blog.md
diff --git a/k8s/clickhouse/chi-logs/logs.yaml b/k8s/clickhouse/chi-logs/logs.yaml
index 7f076d8..69e5786 100644
--- a/k8s/clickhouse/chi-logs/logs.yaml
+++ b/k8s/clickhouse/chi-logs/logs.yaml
@@ -54,7 +54,7 @@ spec:
             - ReadWriteOnce
           resources:
             requests:
-              storage: 800Gi
+              storage: 1200Gi
     serviceTemplates:
       - name: svc-template
         spec:
           nodeSelector:
             intend: monitoring
```

This small update is all it takes to tell the ClickHouse Operator to scale up the disk size.

---

## Monitoring the Operator Logs

After applying the configuration change, the ClickHouse Operator initiates a rolling update of the pods. The operator logs provide an inside look at the progress.

Normally, all pods will be recreated during the reconcilation process.

In the one of the pod log for clickhouse-operator, we can see pod is recreating.

```
clickhouse-operator I1017 15:27:41.310739       1 poller.go:330] pollHostContext():clickhouse-operator/0-1-WAIT
clickhouse-operator I1017 15:27:46.311792       1 cluster.go:84] Run query on: chi-logs-logs-0-1.clickhouse-operator.svc.cluster.local of [chi-logs-logs-0-1.clickhouse-operator.svc.cluster.local]
clickhouse-operator E1017 15:29:16.866848       1 poller.go:335] pollHostContext():clickhouse-operator/0-1-TIMEOUT reached

clickhouse-operator I1017 15:29:16.874777       1 worker.go:1011] updateConfigMap():clickhouse-operator/logs/a521fecd-9ecf-4ff4-aebd-3aceaf29b31a:Update ConfigMap clickhouse-operator/chi-logs-deploy-confd-logs-0-1
clickhouse-operator I1017 15:29:16.960523       1 creator.go:542] getPodTemplate():clickhouse-operator/logs/a521fecd-9ecf-4ff4-aebd-3aceaf29b31a:statefulSet chi-logs-logs-0-1 use custom template clickhouse:23.3
clickhouse-operator I1017 15:29:16.961615       1 worker.go:1212] getStatefulSetStatus():clickhouse-operator/chi-logs-logs-0-1:cur and new StatefulSets ARE DIFFERENT based on labels. Reconcile is required for: clickhouse-operator/chi-logs-logs-0-1
clickhouse-operator I1017 15:29:16.961684       1 worker.go:1342] updateStatefulSet():Update StatefulSet(clickhouse-operator/chi-logs-logs-0-1) - started
clickhouse-operator I1017 15:29:17.706860       1 worker.go:1312] waitConfigMapPropagation():Wait for ConfigMap propagation for 9.223177994s 776.822006ms/10s
clickhouse-operator E1017 15:29:26.941945       1 creator.go:78] updateStatefulSet():StatefulSet.apps "chi-logs-logs-0-1" is invalid: spec: Forbidden: updates to statefulset spec for fields other than 'replicas', 'ordinals', 'template', 'updateStrategy', 'persistentVolumeClaimRetentionPolicy' and 'minReadySeconds' are forbidden
clickhouse-operator E1017 15:29:26.942759       1 creator.go:102] updateStatefulSet():NOT EQUAL: AP item start -------------------------
clickhouse-operator modified spec items: 21
clickhouse-operator ap item path [0]:'.Template.Spec.Containers[0].LivenessProbe.TimeoutSeconds'
clickhouse-operator ap item value[0]:'0'
clickhouse-operator ap item path [1]:'.Template.Spec.Containers[0].ReadinessProbe.TimeoutSeconds'
clickhouse-operator ap item value[1]:'0'
clickhouse-operator ap item path [2]:'.Template.Spec.Containers[0].ReadinessProbe.SuccessThreshold'
clickhouse-operator ap item value[2]:'0'
clickhouse-operator ap item path [3]:'.Template.Spec.RestartPolicy'
clickhouse-operator ap item value[3]:'""'
clickhouse-operator ap item path [4]:'.Template.Spec.DNSPolicy'
clickhouse-operator ap item value[4]:'""'
clickhouse-operator ap item path [5]:'.Template.Spec.Containers[0].Ports[0].Protocol'
clickhouse-operator ap item value[5]:'""'
clickhouse-operator ap item path [6]:'.Template.Spec.Containers[0].LivenessProbe.Handler.HTTPGet.Scheme'
clickhouse-operator ap item value[6]:'""'
clickhouse-operator ap item path [7]:'.VolumeClaimTemplates[0].ObjectMeta.Annotations'
clickhouse-operator ap item value[7]:'map[string]string{
clickhouse-operator }'
clickhouse-operator ap item path [8]:'.VolumeClaimTemplates[0].Status.Phase'
clickhouse-operator ap item value[8]:'""'
clickhouse-operator ap item path [9]:'.Template.ObjectMeta.Annotations'
clickhouse-operator ap item value[9]:'map[string]string{
clickhouse-operator }'
clickhouse-operator ap item path [10]:'.Template.Spec.Containers[0].Ports[2].Protocol'
clickhouse-operator ap item value[10]:'""'
clickhouse-operator ap item path [11]:'.Template.Spec.Containers[0].LivenessProbe.SuccessThreshold'
clickhouse-operator ap item value[11]:'0'
clickhouse-operator ap item path [12]:'.Template.Spec.Containers[0].ReadinessProbe.Handler.HTTPGet.Scheme'
clickhouse-operator ap item value[12]:'""'
clickhouse-operator ap item path [13]:'.Template.Spec.Containers[0].ReadinessProbe.FailureThreshold'
clickhouse-operator ap item value[13]:'0'
clickhouse-operator ap item path [14]:'.Template.Spec.Containers[0].TerminationMessagePath'
clickhouse-operator ap item value[14]:'""'
clickhouse-operator ap item path [15]:'.Template.Spec.Containers[0].Ports[1].Protocol'
clickhouse-operator ap item value[15]:'""'
clickhouse-operator ap item path [16]:'.Template.Spec.Containers[0].TerminationMessagePolicy'
clickhouse-operator ap item value[16]:'""'
clickhouse-operator ap item path [17]:'.Template.Spec.Containers[0].ImagePullPolicy'
clickhouse-operator ap item value[17]:'""'
clickhouse-operator ap item path [18]:'.Template.Spec.SecurityContext'
clickhouse-operator ap item value[18]:'nil'
clickhouse-operator ap item path [19]:'.Template.Spec.SchedulerName'
clickhouse-operator ap item value[19]:'""'
clickhouse-operator ap item path [20]:'.VolumeClaimTemplates[0].Spec.Resources.Requests["storage"].i.value'
clickhouse-operator ap item value[20]:'1288490188800'
clickhouse-operator AP item end -------------------------
clickhouse-operator I1017 15:29:26.942798       1 worker.go:1369] updateStatefulSet():Update StatefulSet(clickhouse-operator/chi-logs-logs-0-1) switch from Update to Recreate
clickhouse-operator E1017 15:29:27.008616       1 worker.go:1373] updateStatefulSet():Update StatefulSet(clickhouse-operator/chi-logs-logs-0-1) - failed with error
clickhouse-operator ---
clickhouse-operator StatefulSet.apps "chi-logs-logs-0-1" is invalid: spec: Forbidden: updates to statefulset spec for fields other than 'replicas', 'ordinals', 'template', 'updateStrategy', 'persistentVolumeClaimRetentionPolicy' and 'minReadySeconds' are forbidden
clickhouse-operator --
clickhouse-operator Continue with recreate
clickhouse-operator I1017 15:29:27.009393       1 deleter.go:126] deleteStatefulSet():clickhouse-operator/chi-logs-logs-0-1
clickhouse-operator I1017 15:29:32.040669       1 poller.go:224] pollStatefulSet():clickhouse-operator/chi-logs-logs-0-1:OK  :ObservedGeneration:2 Replicas:1 ReadyReplicas:1 CurrentReplicas:0 UpdatedReplicas:0 CurrentRevision:chi-logs-logs-0-1-547db7557f UpdateRevision:chi-logs-logs-0-1-547db7557f
clickhouse-operator I1017 15:29:37.052566       1 poller.go:224] pollStatefulSet():clickhouse-operator/chi-logs-logs-0-1:OK  :ObservedGeneration:2 Replicas:0 ReadyReplicas:0 CurrentReplicas:0 UpdatedReplicas:0 CurrentRevision:chi-logs-logs-0-1-547db7557f UpdateRevision:chi-logs-logs-0-1-547db7557f
clickhouse-operator I1017 15:29:37.065697       1 deleter.go:154] OK delete StatefulSet clickhouse-operator/chi-logs-logs-0-1

clickhouse-operator ap item value[13]:'0'
clickhouse-operator ap item path [14]:'.Template.Spec.Containers[0].TerminationMessagePath'
clickhouse-operator ap item value[14]:'""'
clickhouse-operator ap item path [15]:'.Template.Spec.Containers[0].Ports[1].Protocol'
clickhouse-operator ap item value[15]:'""'
clickhouse-operator ap item path [16]:'.Template.Spec.Containers[0].TerminationMessagePolicy'
clickhouse-operator ap item value[16]:'""'
clickhouse-operator ap item path [17]:'.Template.Spec.Containers[0].ImagePullPolicy'
clickhouse-operator ap item value[17]:'""'
clickhouse-operator ap item path [18]:'.Template.Spec.SecurityContext'
clickhouse-operator ap item value[18]:'nil'
clickhouse-operator ap item path [19]:'.Template.Spec.SchedulerName'
clickhouse-operator ap item value[19]:'""'
clickhouse-operator ap item path [20]:'.VolumeClaimTemplates[0].Spec.Resources.Requests["storage"].i.value'
clickhouse-operator ap item value[20]:'1288490188800'
clickhouse-operator AP item end -------------------------
clickhouse-operator I1017 15:29:26.942798       1 worker.go:1369] updateStatefulSet():Update StatefulSet(clickhouse-operator/chi-logs-logs-0-1) switch from Update to Recreate
clickhouse-operator E1017 15:29:27.008616       1 worker.go:1373] updateStatefulSet():Update StatefulSet(clickhouse-operator/chi-logs-logs-0-1) - failed with error
clickhouse-operator ---
clickhouse-operator StatefulSet.apps "chi-logs-logs-0-1" is invalid: spec: Forbidden: updates to statefulset spec for fields other than 'replicas', 'ordinals', 'template', 'updateStrategy', 'persistentVolumeClaimRetentionPolicy' and 'minReadySeconds' are forbidden
clickhouse-operator --
clickhouse-operator Continue with recreate
clickhouse-operator I1017 15:29:27.009393       1 deleter.go:126] deleteStatefulSet():clickhouse-operator/chi-logs-logs-0-1
clickhouse-operator I1017 15:29:32.040669       1 poller.go:224] pollStatefulSet():clickhouse-operator/chi-logs-logs-0-1:OK  :ObservedGeneration:2 Replicas:1 ReadyReplicas:1 CurrentReplicas:0 UpdatedReplicas:0 CurrentRevision:chi-logs-logs-0-1-547db7557f UpdateRevision:chi-logs-logs-0-1-547db7557f
clickhouse-operator I1017 15:29:37.052566       1 poller.go:224] pollStatefulSet():clickhouse-operator/chi-logs-logs-0-1:OK  :ObservedGeneration:2 Replicas:0 ReadyReplicas:0 CurrentReplicas:0 UpdatedReplicas:0 CurrentRevision:chi-logs-logs-0-1-547db7557f UpdateRevision:chi-logs-logs-0-1-547db7557f
clickhouse-operator I1017 15:29:37.065697       1 deleter.go:154] OK delete StatefulSet clickhouse-operator/chi-logs-logs-0-1
clickhouse-operator I1017 15:29:52.078823       1 poller.go:98] cache synced
clickhouse-operator I1017 15:29:52.120950       1 worker.go:1247] createStatefulSet():Create StatefulSet clickhouse-operator/chi-logs-logs-0-1 - started
clickhouse-operator I1017 15:29:52.186657       1 creator.go:35] createStatefulSet()
clickhouse-operator I1017 15:29:52.186677       1 creator.go:44] Create StatefulSet clickhouse-operator/chi-logs-logs-0-1
clickhouse-operator I1017 15:29:57.222364       1 poller.go:224] pollStatefulSet():clickhouse-operator/chi-logs-logs-0-1:OK  :ObservedGeneration:1 Replicas:1 ReadyReplicas:0 CurrentReplicas:1 UpdatedReplicas:1 CurrentRevision:chi-logs-logs-0-1-547db7557f UpdateRevision:chi-logs-logs-0-1-547db7557f
clickhouse-operator I1017 15:30:02.256953       1 worker.go:324] clickhouse-operator/logs/391a2022-e3d8-44b5-ab7f-10c09f8c0e19:IPs of the CHI-1 [10.120.132.197 10.120.143.45 10.120.143.52 10.120.143.10]
clickhouse-operator I1017 15:30:02.267330       1 worker.go:328] clickhouse-operator/logs/d14bf645-d654-4e19-b377-5cc6bfcc582a:Update users IPS-1
clickhouse-operator I1017 15:30:02.279076       1 worker.go:1011] updateConfigMap():clickhouse-operator/logs/d14bf645-d654-4e19-b377-5cc6bfcc582a:Update ConfigMap clickhouse-operator/chi-logs-common-usersd
```

While the output is long, we can see one of the pods(`clickhouse-operator/chi-logs-logs-0-1`) is
deleted and recreated Successfully.

```bash
# delete
clickhouse-operator I1017 15:29:37.065697       1 deleter.go:154] OK delete StatefulSet clickhouse-operator/chi-logs-logs-0-1

# recreated
clickhouse-operator I1017 15:29:52.186677       1 creator.go:44] Create StatefulSet clickhouse-operator/chi-logs-logs-0-1
```

Let's see what happened in the background.

In the above snippet, the operator first attempts to update the StatefulSet but then opts for a recreate strategy by deleting the affected pod.

# kubernetes pods

We can also see from k9s UI that one of the pods is recreating.

```
 Context: arn:aws-cn:eks:cn-northwest-2:321321321321:cluster/log-cluster      <0> all                   <a>      Attach     <l>       Logs               <y> YAML               ____  __.________
 Cluster: arn:aws-cn:eks:cn-northwest-1:321321321321:cluster/log-cluster      <1> clickhouse-operator   <ctrl-d> Delete     <p>       Logs Previous                            |    |/ _/   __   \______
 User:    arn:aws-cn:eks:cn-northwest-1:321321321321:cluster/log-cluster      <2> default               <d>      Describe   <shift-f> Port-Forward                             |      < \____    /  ___/
 K9s Rev: v0.27.4                                                                                <e>      Edit       <s>       Shell                                    |    |  \   /    /\___ \
 K8s Rev: v1.27.4-eks-2d98532                                                                    <?>      Help       <n>       Show Node                                |____|__ \ /____//____  >
 CPU:     5%                                                                                     <ctrl-k> Kill       <f>       Show PortForward                                 \/            \/
 MEM:     29%
┌───────────────────────────────────────────────────────────────────────────────── Pods(clickhouse-operator)[5] ─────────────────────────────────────────────────────────────────────────────────┐
│ NAME↑                                 PF READY RESTARTS STATUS         CPU   MEM  %CPU/R  %CPU/L  %MEM/R  %MEM/L IP               NODE                                                AGE      │
│ chi-logs-logs-0-0-0                   ●  0/1Δ         0 Terminating    766  1436     n/a     n/a     n/a     n/a n/aΔ             ip-10-120-130-142.cn-northwest-1.compute.internal   90m      │
│ chi-logs-logs-0-1-0                   ●  1/1          0 Running        823  1307     n/a     n/a     n/a     n/a 10.120.143.45    ip-10-120-130-142.cn-northwest-1.compute.internal   10m      │
│ chi-logs-logs-1-0-0                   ●  1/1          0 Running        947  1404     n/a     n/a     n/a     n/a 10.120.135.149   ip-10-120-130-142.cn-northwest-1.compute.internal   7m17s    │
│ chi-logs-logs-1-1-0                   ●  1/1          0 Running        766  1436     n/a     n/a     n/a     n/a 10.120.140.217   ip-10-120-130-142.cn-northwest-1.compute.internal   90m      │
│ clickhouse-operator-55959cbf5d-n5swv  ●  2/2          0 Running          6    33     n/a     n/a     n/a     n/a 10.120.152.233   ip-10-120-158-240.cn-northwest-1.compute.internal   30d      │
```

Notice that the pod `chi-logs-logs-0-0-0` is being terminated and will be recreated.

# kubernetes Events

We can find some evidence of pod recreating from kubernetes events.

```bash
k get events -n clickhouse-operator -w
```

```
2m15s       Normal    SuccessfulDelete             statefulset/chi-logs-logs-0-0                                         delete Pod chi-logs-logs-0-0-0 in StatefulSet chi-logs-logs-0-0 successful
110s        Normal    SuccessfulCreate             statefulset/chi-logs-logs-0-0                                         create Pod chi-logs-logs-0-0-0 in StatefulSet chi-logs-logs-0-0 successful
24m         Normal    Killing                      pod/chi-logs-logs-1-0-0                                               Stopping container clickhouse-pod
24m         Normal    Scheduled                    pod/chi-logs-logs-1-0-0                                               Successfully assigned clickhouse-operator/chi-logs-logs-1-0-0 to ip-10-120-130-142.cn-northwest-1.compute.internal
```

From the events, we can see that the pod `chi-logs-logs-0-0-0` is being terminated and recreated again.

Below is all the events of scaling disk from 800G to 1200G.

```
LAST SEEN   TYPE      REASON                       OBJECT                                                                MESSAGE
2m15s       Normal    Killing                      pod/chi-logs-logs-0-0-0                                               Stopping container clickhouse-pod
2m10s       Warning   Unhealthy                    pod/chi-logs-logs-0-0-0                                               Readiness probe failed: Get "http://10.120.142.251:8123/ping": dial tcp 10.120.142.251:8123: connect: connection refused
110s        Normal    Scheduled                    pod/chi-logs-logs-0-0-0                                               Successfully assigned clickhouse-operator/chi-logs-logs-0-0-0 to ip-10-120-130-142.cn-northwest-1.compute.internal
108s        Normal    SuccessfulAttachVolume       pod/chi-logs-logs-0-0-0                                               AttachVolume.Attach succeeded for volume "pvc-54910269-0c08-4170-8c04-0aba526dcf7e"
102s        Normal    FileSystemResizeSuccessful   pod/chi-logs-logs-0-0-0                                               MountVolume.NodeExpandVolume succeeded for volume "pvc-54910269-0c08-4170-8c04-0aba526dcf7e" ip-10-120-130-142.cn-northwest-1.compute.internal
101s        Normal    Pulled                       pod/chi-logs-logs-0-0-0                                               Container image "321321321321.dkr.ecr.cn-northwest-1.amazonaws.com.cn/clickhouse/clickhouse-server:23.3" already present on machine
101s        Normal    Created                      pod/chi-logs-logs-0-0-0                                               Created container clickhouse-pod
101s        Normal    Started                      pod/chi-logs-logs-0-0-0                                               Started container clickhouse-pod
2m15s       Normal    SuccessfulDelete             statefulset/chi-logs-logs-0-0                                         delete Pod chi-logs-logs-0-0-0 in StatefulSet chi-logs-logs-0-0 successful
110s        Normal    SuccessfulCreate             statefulset/chi-logs-logs-0-0                                         create Pod chi-logs-logs-0-0-0 in StatefulSet chi-logs-logs-0-0 successful
24m         Normal    Killing                      pod/chi-logs-logs-1-0-0                                               Stopping container clickhouse-pod
24m         Normal    Scheduled                    pod/chi-logs-logs-1-0-0                                               Successfully assigned clickhouse-operator/chi-logs-logs-1-0-0 to ip-10-120-130-142.cn-northwest-1.compute.internal
24m         Normal    Pulled                       pod/chi-logs-logs-1-0-0                                               Container image "321321321321.dkr.ecr.cn-northwest-1.amazonaws.com.cn/clickhouse/clickhouse-server:23.3" already present on machine
24m         Normal    Created                      pod/chi-logs-logs-1-0-0                                               Created container clickhouse-pod
24m         Normal    Started                      pod/chi-logs-logs-1-0-0                                               Started container clickhouse-pod
23m         Normal    Killing                      pod/chi-logs-logs-1-0-0                                               Stopping container clickhouse-pod
23m         Normal    Scheduled                    pod/chi-logs-logs-1-0-0                                               Successfully assigned clickhouse-operator/chi-logs-logs-1-0-0 to ip-10-120-130-142.cn-northwest-1.compute.internal
23m         Normal    Pulled                       pod/chi-logs-logs-1-0-0                                               Container image "321321321321.dkr.ecr.cn-northwest-1.amazonaws.com.cn/clickhouse/clickhouse-server:23.3" already present on machine
23m         Normal    Created                      pod/chi-logs-logs-1-0-0                                               Created container clickhouse-pod
23m         Normal    Started                      pod/chi-logs-logs-1-0-0                                               Started container clickhouse-pod
24m         Normal    SuccessfulCreate             statefulset/chi-logs-logs-1-0                                         create Pod chi-logs-logs-1-0-0 in StatefulSet chi-logs-logs-1-0 successful
23m         Warning   RecreatingFailedPod          statefulset/chi-logs-logs-1-0                                         StatefulSet clickhouse-operator/chi-logs-logs-1-0 is recreating failed Pod chi-logs-logs-1-0-0
23m         Normal    SuccessfulDelete             statefulset/chi-logs-logs-1-0                                         delete Pod chi-logs-logs-1-0-0 in StatefulSet chi-logs-logs-1-0 successful
24m         Warning   FailedDelete                 statefulset/chi-logs-logs-1-0                                         delete Pod chi-logs-logs-1-0-0 in StatefulSet chi-logs-logs-1-0 failed error: pods "chi-logs-logs-1-0-0" not found
74s         Info      ReconcileCompleted           clickhouseinstallation/logs                                           Reconcile Host 0-0 completed
110s        Info      CreateStarted                clickhouseinstallation/logs                                           Create StatefulSet clickhouse-operator/chi-logs-logs-0-0 - started
65s         Info      UpdateCompleted              clickhouseinstallation/logs                                           Update ConfigMap clickhouse-operator/chi-logs-common-usersd
72s         Info      UpdateCompleted              clickhouseinstallation/logs                                           Update ConfigMap clickhouse-operator/chi-logs-common-usersd
2m26s       Info      CreateStarted                clickhouseinstallation/logs                                           Update StatefulSet(clickhouse-operator/chi-logs-logs-0-0) - started
3m33s       Info      UpdateCompleted              clickhouseinstallation/logs                                           Update ConfigMap clickhouse-operator/chi-logs-common-configd
84s         Info      UpdateCompleted              clickhouseinstallation/logs                                           Update Service clickhouse-operator/chi-logs-logs-0-0
100s        Info      UpdateCompleted              clickhouseinstallation/logs                                           Update ConfigMap clickhouse-operator/chi-logs-common-usersd
23m         Info      UpdateCompleted              clickhouseinstallation/logs                                           Update ConfigMap clickhouse-operator/chi-logs-common-usersd
66s         Info      UpdateCompleted              clickhouseinstallation/logs                                           Update ConfigMap clickhouse-operator/chi-logs-common-configd
74s         Info      UpdateCompleted              clickhouseinstallation/logs                                           Update ConfigMap clickhouse-operator/chi-logs-common-configd
3m35s       Info      ReconcileStarted             clickhouseinstallation/logs                                           Reconcile Host 0-0 started
3m40s       Info      ReconcileStarted             clickhouseinstallation/logs                                           reconcile started
24m         Info      UpdateCompleted              clickhouseinstallation/logs                                           Update ConfigMap clickhouse-operator/chi-logs-common-usersd
24m         Info      UpdateCompleted              clickhouseinstallation/logs                                           Update ConfigMap clickhouse-operator/chi-logs-common-usersd
23m         Info      UpdateCompleted              clickhouseinstallation/logs                                           Update ConfigMap clickhouse-operator/chi-logs-common-usersd
70s         Info      UpdateCompleted              clickhouseinstallation/logs                                           Update Service clickhouse-operator/clickhouse-logs
85s         Info      CreateCompleted              clickhouseinstallation/logs                                           Create StatefulSet clickhouse-operator/chi-logs-logs-0-0 - completed
3m32s       Info      UpdateCompleted              clickhouseinstallation/logs                                           Update ConfigMap clickhouse-operator/chi-logs-common-usersd
72s         Info      ProgressHostsCompleted       clickhouseinstallation/logs                                           ProgressHostsCompleted: 1 of 4
3m36s       Info      UpdateCompleted              clickhouseinstallation/logs                                           Update ConfigMap clickhouse-operator/chi-logs-common-usersd
83s         Info      CreateStarted                clickhouseinstallation/logs                                           Adding tables on shard/host:0/0 cluster:logs
68s         Info      ReconcileStarted             clickhouseinstallation/logs                                           Reconcile Host 0-1 started
2m26s       Info      UpdateCompleted              clickhouseinstallation/logs                                           Update ConfigMap clickhouse-operator/chi-logs-deploy-confd-logs-0-0
3m37s       Info      UpdateCompleted              clickhouseinstallation/logs                                           Update ConfigMap clickhouse-operator/chi-logs-common-configd
2m16s       Info      UpdateInProgress             clickhouseinstallation/logs                                           Update StatefulSet(clickhouse-operator/chi-logs-logs-0-0) switch from Update to Recreate
2m3s        Normal    Resizing                     persistentvolumeclaim/data-volumeclaim-template-chi-logs-logs-0-0-0   External resizer is resizing volume pvc-54910269-0c08-4170-8c04-0aba526dcf7e
2m26s       Warning   ExternalExpanding            persistentvolumeclaim/data-volumeclaim-template-chi-logs-logs-0-0-0   waiting for an external controller to expand this PVC
2m16s       Warning   VolumeResizeFailed           persistentvolumeclaim/data-volumeclaim-template-chi-logs-logs-0-0-0   resize volume "pvc-54910269-0c08-4170-8c04-0aba526dcf7e" by resizer "ebs.csi.aws.com" failed: rpc error: code = DeadlineExceeded desc = context deadline exceeded
2m5s        Warning   VolumeResizeFailed           persistentvolumeclaim/data-volumeclaim-template-chi-logs-logs-0-0-0   resize volume "pvc-54910269-0c08-4170-8c04-0aba526dcf7e" by resizer "ebs.csi.aws.com" failed: rpc error: code = Internal desc = Could not resize volume "vol-0f696cea1221fa93f": context cancelled
2m          Normal    FileSystemResizeRequired     persistentvolumeclaim/data-volumeclaim-template-chi-logs-logs-0-0-0   Require file system resize of volume on node
102s        Normal    FileSystemResizeSuccessful   persistentvolumeclaim/data-volumeclaim-template-chi-logs-logs-0-0-0   MountVolume.NodeExpandVolume succeeded for volume "pvc-54910269-0c08-4170-8c04-0aba526dcf7e" ip-10-120-130-142.cn-northwest-1.compute.internal

0s          Info      UpdateCompleted              clickhouseinstallation/logs                                           Update ConfigMap clickhouse-operator/chi-logs-deploy-confd-logs-0-1
0s          Info      CreateStarted                clickhouseinstallation/logs                                           Update StatefulSet(clickhouse-operator/chi-logs-logs-0-1) - started
0s          Normal    Resizing                     persistentvolumeclaim/data-volumeclaim-template-chi-logs-logs-0-1-0   External resizer is resizing volume pvc-f9842a2c-94a6-4079-956b-7e04e7df1a77
0s          Warning   ExternalExpanding            persistentvolumeclaim/data-volumeclaim-template-chi-logs-logs-0-1-0   waiting for an external controller to expand this PVC
0s          Info      UpdateInProgress             clickhouseinstallation/logs                                           Update StatefulSet(clickhouse-operator/chi-logs-logs-0-1) switch from Update to Recreate
0s          Warning   VolumeResizeFailed           persistentvolumeclaim/data-volumeclaim-template-chi-logs-logs-0-1-0   resize volume "pvc-f9842a2c-94a6-4079-956b-7e04e7df1a77" by resizer "ebs.csi.aws.com" failed: rpc error: code = Internal desc = Could not resize volume "vol-0699b47667a442b1a": context cancelled
0s          Normal    Resizing                     persistentvolumeclaim/data-volumeclaim-template-chi-logs-logs-0-1-0   External resizer is resizing volume pvc-f9842a2c-94a6-4079-956b-7e04e7df1a77
0s          Normal    Killing                      pod/chi-logs-logs-0-1-0                                               Stopping container clickhouse-pod
0s          Normal    SuccessfulDelete             statefulset/chi-logs-logs-0-1                                         delete Pod chi-logs-logs-0-1-0 in StatefulSet chi-logs-logs-0-1 successful
0s          Warning   Unhealthy                    pod/chi-logs-logs-0-1-0                                               Readiness probe failed: Get "http://10.120.128.145:8123/ping": dial tcp 10.120.128.145:8123: connect: connection refused
0s          Warning   Unhealthy                    pod/chi-logs-logs-0-1-0                                               Readiness probe failed: Get "http://10.120.128.145:8123/ping": dial tcp 10.120.128.145:8123: connect: connection refused
0s          Warning   VolumeResizeFailed           persistentvolumeclaim/data-volumeclaim-template-chi-logs-logs-0-1-0   resize volume "pvc-f9842a2c-94a6-4079-956b-7e04e7df1a77" by resizer "ebs.csi.aws.com" failed: rpc error: code = Internal desc = Could not resize volume "vol-0699b47667a442b1a": context cancelled
1s          Normal    Resizing                     persistentvolumeclaim/data-volumeclaim-template-chi-logs-logs-0-1-0   External resizer is resizing volume pvc-f9842a2c-94a6-4079-956b-7e04e7df1a77
0s          Normal    FileSystemResizeRequired     persistentvolumeclaim/data-volumeclaim-template-chi-logs-logs-0-1-0   Require file system resize of volume on node
0s          Info      CreateStarted                clickhouseinstallation/logs                                           Create StatefulSet clickhouse-operator/chi-logs-logs-0-1 - started
0s          Normal    SuccessfulCreate             statefulset/chi-logs-logs-0-1                                         create Pod chi-logs-logs-0-1-0 in StatefulSet chi-logs-logs-0-1 successful
0s          Normal    Scheduled                    pod/chi-logs-logs-0-1-0                                               Successfully assigned clickhouse-operator/chi-logs-logs-0-1-0 to ip-10-120-130-142.cn-northwest-1.compute.internal
0s          Normal    SuccessfulAttachVolume       pod/chi-logs-logs-0-1-0                                               AttachVolume.Attach succeeded for volume "pvc-f9842a2c-94a6-4079-956b-7e04e7df1a77"
0s          Normal    FileSystemResizeSuccessful   pod/chi-logs-logs-0-1-0                                               MountVolume.NodeExpandVolume succeeded for volume "pvc-f9842a2c-94a6-4079-956b-7e04e7df1a77" ip-10-120-130-142.cn-northwest-1.compute.internal
0s          Normal    FileSystemResizeSuccessful   persistentvolumeclaim/data-volumeclaim-template-chi-logs-logs-0-1-0   MountVolume.NodeExpandVolume succeeded for volume "pvc-f9842a2c-94a6-4079-956b-7e04e7df1a77" ip-10-120-130-142.cn-northwest-1.compute.internal
0s          Normal    FileSystemResizeSuccessful   pod/chi-logs-logs-0-1-0                                               MountVolume.NodeExpandVolume succeeded for volume "pvc-f9842a2c-94a6-4079-956b-7e04e7df1a77" ip-10-120-130-142.cn-northwest-1.compute.internal
0s          Normal    FileSystemResizeSuccessful   persistentvolumeclaim/data-volumeclaim-template-chi-logs-logs-0-1-0   MountVolume.NodeExpandVolume succeeded for volume "pvc-f9842a2c-94a6-4079-956b-7e04e7df1a77" ip-10-120-130-142.cn-northwest-1.compute.internal
0s          Normal    Pulled                       pod/chi-logs-logs-0-1-0                                               Container image "321321321321.dkr.ecr.cn-northwest-1.amazonaws.com.cn/clickhouse/clickhouse-server:23.3" already present on machine
0s          Normal    Created                      pod/chi-logs-logs-0-1-0                                               Created container clickhouse-pod
0s          Normal    Started                      pod/chi-logs-logs-0-1-0                                               Started container clickhouse-pod
0s          Info      UpdateCompleted              clickhouseinstallation/logs                                           Update ConfigMap clickhouse-operator/chi-logs-common-usersd
0s          Warning   Unhealthy                    pod/chi-logs-logs-0-1-0                                               Readiness probe failed: Get "http://10.120.143.45:8123/ping": dial tcp 10.120.143.45:8123: connect: connection refused
0s          Warning   Unhealthy                    pod/chi-logs-logs-0-1-0                                               Readiness probe failed: Get "http://10.120.143.45:8123/ping": dial tcp 10.120.143.45:8123: connect: connection refused
0s          Warning   Unhealthy                    pod/chi-logs-logs-0-1-0                                               Readiness probe failed: Get "http://10.120.143.45:8123/ping": dial tcp 10.120.143.45:8123: connect: connection refused
0s          Warning   Unhealthy                    pod/chi-logs-logs-0-1-0                                               Readiness probe failed: Get "http://10.120.143.45:8123/ping": dial tcp 10.120.143.45:8123: connect: connection refused
0s          Warning   Unhealthy                    pod/chi-logs-logs-0-1-0                                               Readiness probe failed: Get "http://10.120.143.45:8123/ping": dial tcp 10.120.143.45:8123: connect: connection refused
0s          Warning   Unhealthy                    pod/chi-logs-logs-0-1-0                                               Readiness probe failed: Get "http://10.120.143.45:8123/ping": dial tcp 10.120.143.45:8123: connect: connection refused
0s          Warning   Unhealthy                    pod/chi-logs-logs-0-1-0                                               Readiness probe failed: Get "http://10.120.143.45:8123/ping": dial tcp 10.120.143.45:8123: connect: connection refused
0s          Warning   Unhealthy                    pod/chi-logs-logs-0-1-0                                               Readiness probe failed: Get "http://10.120.143.45:8123/ping": dial tcp 10.120.143.45:8123: connect: connection refused
lk0s          Warning   Unhealthy                  pod/chi-logs-logs-0-1-0                                               Readiness probe failed: Get "http://10.120.143.45:8123/ping": dial tcp 10.120.143.45:8123: connect: connection refused
0s          Warning   Unhealthy                    pod/chi-logs-logs-0-1-0                                               Readiness probe failed: Get "http://10.120.143.45:8123/ping": dial tcp 10.120.143.45:8123: connect: connection refused
0s          Warning   Unhealthy                    pod/chi-logs-logs-0-1-0                                               Readiness probe failed: Get "http://10.120.143.45:8123/ping": dial tcp 10.120.143.45:8123: connect: connection refused
0s          Warning   Unhealthy                    pod/chi-logs-logs-0-1-0                                               Readiness probe failed: Get "http://10.120.143.45:8123/ping": dial tcp 10.120.143.45:8123: connect: connection refused


0s          Info      CreateCompleted              clickhouseinstallation/logs                                           Create StatefulSet clickhouse-operator/chi-logs-logs-0-1 - completed
0s          Info      UpdateCompleted              clickhouseinstallation/logs                                           Update Service clickhouse-operator/chi-logs-logs-0-1
0s          Info      CreateStarted                clickhouseinstallation/logs                                           Adding tables on shard/host:0/1 cluster:logs
0s          Info      UpdateCompleted              clickhouseinstallation/logs                                           Update ConfigMap clickhouse-operator/chi-logs-common-configd
0s          Info      ReconcileCompleted           clickhouseinstallation/logs                                           Reconcile Host 0-1 completed
0s          Info      ProgressHostsCompleted       clickhouseinstallation/logs                                           ProgressHostsCompleted: 2 of 4
1s          Info      UpdateCompleted              clickhouseinstallation/logs                                           Update ConfigMap clickhouse-operator/chi-logs-common-usersd
0s          Info      ReconcileStarted             clickhouseinstallation/logs                                           Reconcile Host 1-0 started
0s          Info      UpdateCompleted              clickhouseinstallation/logs                                           Update ConfigMap clickhouse-operator/chi-logs-common-configd
```

From the events, we can see that all the pods are rolling out.

# Conclusion

In this blog post, we discussed the process of extending disk size in a ClickHouse cluster using the ClickHouse Operator. We explored the necessary configuration changes, observed the logs during the scaling process, and reviewed the Kubernetes events that indicate pod recreation. This knowledge is crucial for maintaining optimal performance in your ClickHouse deployments.
