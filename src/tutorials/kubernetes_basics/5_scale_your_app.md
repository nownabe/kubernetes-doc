スケール
========

# 複数インスタンスの起動

https://kubernetes.io/docs/tutorials/kubernetes-basics/scale-intro/

## 目標
* kubectlを使ったスケール

## アプリケーションのスケーリング
以前の章で[Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)を作成し、[Service](https://kubernetes.io/docs/concepts/services-networking/service/)を使って公開しました。
Deploymentはアプリケーションのために1つだけPodを作成しました。
トラフィックが増えた時、アプリケーションを稼働しつづけるためにスケールする必要があります。

**スケーリング**はDeploymentのReplicaの数を変えることで達成できます。

## スケーリング概要
![1](https://kubernetes.io/docs/tutorials/kubernetes-basics/public/images/module_05_scaling1.svg)
![2](https://kubernetes.io/docs/tutorials/kubernetes-basics/public/images/module_05_scaling2.svg)

Deploymentのスケールアップ[^1]は利用可能なりソースを使って新しいPodが作成されることとNodeにスケジュールされることを約束します。
スケールダウンは期待される状態までPodの数を減らします。
KubernetesはPodの[オートスケール](http://kubernetes.io/docs/user-guide/horizontal-pod-autoscaling/)もサポートしていますが、このチュートリアルでは説明しません。
ゼロにスケールすることもできますが、指定されたDeploymentのすべてのPodが終了します。

アプリケーションの複数インスタンスの起動はすべてにトラフィックが分散されるような方法が必要です。
Serviceは公開されたDeploymentのすべてのPodにネットワークトラフィックを分散することが可能なロードバランサを統合しています。
Serviceは起動中のPodのエンドポイントを監視し、生きているPodにのみトラフィックを送信します。

一度あるアプリケーションを複数インスタンスで実行したら、ダウンタイムなしでのローリングアップデートが可能です。
次章で扱います。

[^1]: Podのスケールアウト

# インタラクティブチュートリアル

https://kubernetes.io/docs/tutorials/kubernetes-basics/scale-interactive/

## Step 1: Deploymentのスケーリング

Deploymentを一覧表示するために`get deployments`を使います。

```
kubectl get deployments
```

1つのPodがあることを確認できます。

結果の意味はそれぞれ次のようになります。

* `DESIRED` - 設定されたReplica数
* `CURRENT` - 現在起動中のReplica数
* `UP-TO-DATE` - 設定された状態と一致させるために変更されたReplica数
* `AVAILABLE` - 実際に利用可能なReplica数

次に、Deploymentを4つのReplicaにスケールさせましょう。
`kubectl scale`コマンドを使います。
引数にはDeploymentの種類と名前、期待するReplica数をとります。

```
kubectl scale deployments/kubernetes-bootcamp --replicas=4
```

再び`get deployments`を使ってDeploymentの一覧を表示してください。

```
kubectl get deployments
```

変更が適用され、アプリケーションに4つのインスタンスがあります。
Podの数を確認してみましょう。

```
kubectl get pods -o wide
```

異なるIPの4つのPodが確認できます。
変更はDeploymentのイベントログに登録されています。
describeコマンドで確認してみましょう。

```
kubectl describe deployments/kubernetes-bootcamp
```

## Step 2: ロードバランシング

Serviceがトラフィックを分散させているか確認してみましょう。
公開されているIPとポートを確認するために前章で学んだdescribe serviceを使います。

```
kubectl describe services/kubernetes-bootcamp
```

NODE_PORT環境変数にNodeポートを格納しましょう。

```
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports0).nodePort}}')
echo NODE_PORT=$NODE_PORT
```

何度か`curl`コマンドでリクエストしてみてください。

```
curl host01:$NODE_PORT
```

毎回異なるPodにアクセスしています。負荷分散されています。

## Step 3: スケールダウン

2つのReplicaになるようにスケールダウンしましょう。
再び`scale`コマンドを使います。

```
kubectl scale deployments/kubernetes-bootcamp --replicas=2
```

変更が適用されたかどうか確認するため`get deployments`コマンドでDeploymentの一覧を表示しましょう。

```
kubectl get deployments
```

Replica数が2に減っています。
`get pods`でPod一覧を表示して確認しましょう。

```
kubectl get pods -o wide
```

2つのPodが終了したことを確認できます。
