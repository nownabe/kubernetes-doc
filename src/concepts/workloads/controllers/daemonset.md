DaemonSet
=========

# What is a DaemonSet?

*DaemonSet*はすべて(またはいくつかの)NodeでPodのコピーを実行することを保証します。
NodeがClusterに追加されると、Podはそれらに追加されます。
NodeがClusterから削除されると、PodはGarbage Collectionされます。
DaemonSetを削除すると作成されたPodも削除されます。

DaemonSetの典型的な用途は

* `glusterd`や`ceph`などのストレージデーモンを各ノードで実行
* `fluentd`や`logstash`などのログ収集デーモンを各ノードで実行
* Prometheus Node Exporter、`collectd`、Datadog agent、New Relic agent、Gangliaの`gmond`などのモニタリングデーモンを各ノードで実行

単純なケースでは全てのノードをカバーするひとつのDaemonSetがそれぞれのタイプのデーモンに使用されます。
より複雑な設定では1つの種類のデーモンに複数のDaemonSetを使用できますが、ハードウェアの種類によって異なるフラグや異なるメモリ、CPU要求と共に用いることになるかもしれません。

# Writing a DaemonSet Spec

## Create a DaemonSet

DaemonSetはYAMLで記述できます。
例えば、下の`daemonset.yaml`は`fluentd-elasticsearch`のDockerイメージを実行するDaemonSetです。

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: k8s.gcr.io/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

* YAMLファイルに基いてDaemonSetを作成します: `kubectl create -f daemonset.yaml`

## Required Fields

他のすべてのKubernetesの設定のように、DaemonSetに`apiVersion`、`kind`、`metadata`フィールドは必要です。
設定ファイルが動作するための一般的な情報です。
[deploying applications](https://kubernetes.io/docs/user-guide/deploying-applications/)、[configuring containers](https://kubernetes.io/docs/tasks/)、[object management using kubectl](https://kubernetes.io/docs/concepts/overview/object-management-kubectl/overview/)を見てください。

DaemonSetは[`.spec`](https://github.com/kubernetes/community/blob/master/contributors/devel/api-conventions.md#spec-and-status)セクションも必要です。

## Pod Template

`.spec.template`は`.spec`の必須項目です。

`.spec.template`は[Podテンプレート](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/#pod-templates)です。
これは`apiVersion`と`kind`を持たずネストされていることを除き[Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/)のスキーマと同一です。

Podに必要なフィールドに加えて、DaemonSetのPodテンプレートは適切なlabelを指定する必要があります。
([pod selector](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/#pod-selector)を見てください。)

DaemonSetのPodテンプレートは[`RestartPolicy`](https://kubernetes.io/docs/user-guide/pod-states)が`Always`である必要があります。
指定がなかった場合はデフォルトで`Always`になります。

## Pod Selector

`.spec.selector`フィールドはPodのセレクタです。
[Job](https://kubernetes.io/docs/concepts/jobs/run-to-completion-finite-workloads/)の`.spec.selector`と同様です。

Kubernetes 1.8から`.spec.template`のラベルと一致するPodセレクタを指定する必要があります。
値が設定されないときにデフォルト値は設定されません。
セレクタのデフォルト値設定は`kubectl apply`と互換性がありませんでした。
また、一度DaemonSetが作成されるとその`.spec.selector`は変更できません。
Podセレクタの変更は意図しないPodの孤立を引き起こし、ユーザを混乱させることがわかりました。

`.spec.selector`は2つのフィールドを持つオブジェクトです。

* `matchLabels` - [ReplicationController](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)の`.spec.selector`と同様です。
* `matchExpressions` - キーの指定によりより洗練されたセレクタの構築を許可します。値一覧とキーと値に紐づくオペレータになります。

両方指定されたときはANDで処理されます。

`.spec.selector`が指定されたとき、必ず`.spec.template.metadata.labels`と一致していなければなりません。
一致していないコンフィグはAPIに拒否されます。

また、通常は直接、他のDaemonSet経由、ReplicaSetのような他のController経由に関わらず、このセレクタと一致するlabelを持つPodを作るべきではありません。
さもなければ、DaemonSetコントローラはそれらのPodがそのDaemonSetによって作られたものと考えます。
Kubernetesはユーザがそうすることを防ぎません。
これをしたくなる1つのケースとしては、違う設定値をもつPodをテストのためにあるノードに手動で作るときです。

# How Daemon Pods are Scheduled

# Communicating with Daemon Pods

# Updating a DaemonSet

# Alternatives to DaemonSet



