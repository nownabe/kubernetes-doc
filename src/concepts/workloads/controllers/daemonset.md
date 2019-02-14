DaemonSet
=========

# What is a DaemonSet?

*DaemonSet*はすべて(またはいくつか)のNodeでPodのコピーを実行することを保証します。
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

通常はPodが実行されるマシンはKubernetesスケジューラに選択されます。
しかし、DaemonSetコントローラによって作られたPodは既にマシンが選択されています。
(Pod作成時に`.spec.nodeName`が指定されていて、スケジューラに無視されます)
それにより、

* DaemonSetコントローラではNodeの[`unschedulable`](https://kubernetes.io/docs/admin/node/#manual-node-administration)フィールドは尊重されません
* DaemonSetコントローラはスケジューラが起動していなくてもPodを作成することができるので、クラスタの起動の助けになります

DaemonSetのPodは[`taintsとtolerations`](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration)を尊重しますが、Podは次のtaintsのための`NoExecute` tolerationsを持ち、`tolerationSeconds`を持たず作成されます。

* `node.kubernetes.io/not-ready`
* `node.alpha.kubernetes.io/unreachable`

これは`TaintBasedEvictions`機能(alpha)が有効なとき、ネットワーク分断のようなNode障害があったときにPodが追い出されないことを保証します。
(`TaintBasedEvictions`が無効のときも追い出されませんが、tolerationsよりもNodeコントローラにハードコードされた振る舞いをします。)

次のtaintsも有効です。

* `node.kubernetes.io/memory-pressure`
* `node.kubernetes.io/disk-pressure`

critical podが有効でDaemonSetのPodがcriticalとラベル付けされているとき、Daemon Podは`node.kubernetes.io/out-of-disk`のための`NoSchedule` tolerationとともに作成されます。

以上すべての`NoSchedule` taintは1.8以降で`TaintNodesByCondition`機能(alpha)が有効のときに作成されます。

`node-role.kubernetes.io/master`の`NoSchedule` tolerationは1.6以降ではデフォルトではないためmaster nodeにスケジュールするために必要です。

# Communicating with Daemon Pods

DaemonSetのPodと通信するためにはいくつか可能な方法があります。

* **Push**: DaemonSetのPodをデータベースのような他のサービスにデータを送信するように設定します。クライアントがありません。
* **NodeIP and Known Port**: DaemonSetのPodは`hostPort`を使えるので、NodeのIPで到達できます。クライアントは何らかの方法でNodeのIPを知り、慣習によりポートをしっています。
* **DNS**: 同じPod selectorで[`headless service`](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)を作り、`endpoints`リソースを使うかDNSのAレコードでDaemonSetを発見します。
* **Service**: 同じPod selectorでServiceを作り、Serviceを使ってランダムなNodeのDaemonに到達できます。(特定のNodeに到達する方法はありません)

# Updating a DaemonSet

Nodeのラベルが変更されると、DaemonSetはただちに新しくマッチするNodeにPodを追加して、新しくマッチしないNodeからPodを削除します。

DaemonSetが作成したPodは変更できます。
しかし、すべてのPodのフィールドを変更できるわけではありません。
また、DaemonSetコントローラは次回Nodeが作成されたとき(同じ名前であっても)元のテンプレートを使います。

DaemonSetを削除できます。
もし`kubeclt`で`--cascade=false`を指定したら、PodはNodeに残されます。
そして別のテンプレートを使って新しいDaemonSetを作成できます。
別のテンプレートの新しいDaemonSetはラベルがマッチするすべての既存のPodを認識します。
Podテンプレートに不一致があっても修正も削除もしません。
Nodeの削除かPodの削除によって新しいPodを作る必要があるでしょう。

Kubernetes 1.6以降ではDaemonSetの[`ローリングアップデート`](https://kubernetes.io/docs/tasks/manage-daemon/update-daemon-set/)ができます。

# Alternatives to DaemonSet

## Init Scripts

直接Daemonプロセスをノード上で起動することでDaemonプロセスを実行できます。
(`init`、`upstartd`、`systemd`を使って)
これは完璧です。
ただし、それらに比べてDaemonSetはいくつかメリットがあります。

* アプリケーションと同じ方法でDaemonの監視とログ管理が可能
* アプリケーションと同様の設定とツール(Pod Templateや`kubectl`)
* Resource limitが設定されたDaemonを実行することでアプリケーションのコンテナとの分離を強化できます。ただし、これはPodではないコンテナを実行することでも達成できます(例えばDockerで直接起動する)

## Bare Pods

特定のノード上に直接Podを作成して実現できます。
ただし、DaemonSetはNode障害や破壊的なNodeメンテナンスやカーネルアップグレードなどの何らかの理由で終了したり削除されたPodを交換します。
この理由により、個別のPodよりDaemonSetを使うべきです。

## Static Pods

Kubeletが監視しているディレクトリにファイルを書くことでPodを作成できます。
これは[`static pods`](https://kubernetes.io/docs/tasks/administer-cluster/static-pod/)と呼ばれています。
DaemonSetと違ってstatic Podはkubectlや他のKubernetes APIクライアントから管理できません。
Static Podはapiserverに依存しないので、クラスタの起動時に必要な場合に便利です。
また、Static Podは将来非推奨になるでしょう。

## Deployments

DaemonSetはPodを作成し、Podが終了を想定しないプロセス(例えばWebサーバやストレージサーバ)を持つという意味で[Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)に似ています。

フロントエンドのようなPodが実行されるホストを正確にコントロールするよりも、ローリングアップデートやレプリカ数のスケールアップ/ダウンが重要なステートレスサービスにはDeploymentを使って下さい。
DaemonSetはすべてあるいは一定のホストで1つのPodのコピーが常に実行されていることや、他のPodの前に起動していることが重要な場合に使って下さい。
