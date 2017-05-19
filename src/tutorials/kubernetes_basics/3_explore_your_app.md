PodとNode
=========

# PodとNode

https://kubernetes.io/docs/tutorials/kubernetes-basics/explore-intro/

## 目標

* KubernetesのPodについて学ぶ
* KubernetesのNodeについて学ぶ
* デプロイされたアプリケーションのトラブルシューティング

## Pod

2章でDeploymentを作ったとき、Kubernetesはアプリケーションをホストするために**Pod**を作成しました。
Podは1個または2個以上のアプリケーションコンテナのグループを表す概念で、Pod内のコンテナ群はいくつかのリソースを共有しています。
共有するリソースは以下です。

* Volumeとしてのストレージ
* ユニークなクラスタのIPアドレスとしてのネットワーキング
* コンテナイメージのバージョンや利用ポートといった、それぞれのコンテナがどう動作するかという情報

Podはアプリケーション特有の"論理ホスト"を形成しており、密結合された異なるアプリケーションコンテナを含むことができます。
For example, a Pod might include both the container with your Node.js app as well as a different container that feeds the data to be published by the Node.js webserver.
Podのコンテナ群はひとつのIPアドレスとポート空間を共有し、いつも一緒にスケジュールされ配置され、同じNode上で起動します。

PodはKubernetesにおける最小の単位です。
KubernetesにDeploymentを作るとき、Deploymentはコンテナを直接作るのではなく、コンテナを内包したPod(群)を作成します。
それぞれのPodはスケジュールされたNodeに紐付けられ、終了する[^1]か削除されるまで存続します。
Node障害の場合は、同一のPod(群)がクラスタ内の他の利用可能なNodeにスケジュールされます。

![Pod](https://kubernetes.io/docs/tutorials/kubernetes-basics/public/images/module_03_pods.svg)

## Node

Podは必ずNode上で実行されます。
NodeはKubernetesのワーカーマシンで、仮想マシンか物理マシンです。
それぞれのNodeはMasterに管理されます。
ひとつのNodeは複数のPodを持てます。
Masterはクラスタ内のノード全体でPodのスケジューリングを自動的に処理します。
Masterの自動スケジューリングはそれぞれのノードの利用可能なりソースを考慮します。

すべてのKubernetes Nodeは少なくとも次のものが起動しています。

* kubelet: MasterとNode間の通信を行うプロセス。マシン上で実行されるコンテナとPodの管理
* コンテナランタイム(Dockerやrkt): レジストリからのコンテナイメージの取得、コンテナの展開、アプリケーションの実行

![Node](https://kubernetes.io/docs/tutorials/kubernetes-basics/public/images/module_03_nodes.svg)

## kubectlのトラブルシューティング
2章でkubectlのCLIを使いました。
この章でもデプロイされたアプリケーションの情報や環境を取得するためにkubectlを使います。
もっとも一般的な操作は次のコマンドで可能です。

* `kubectl get` - リソース一覧
* `kubectl describe` - リソースの詳細情報表示
* `kubectl logs` - Pod内のコンテナのログを表示
* `kubectl exec` - Pod内のコンテナでコマンドを実行

いつアプリケーションがデプロイされたか、今の状態、どこで実行されているか、設定はどうなっているか、といった情報を確認できます。

[^1]: 再起動ポリシーに従って

# インタラクティブチュートリアル
https://kubernetes.io/docs/tutorials/kubernetes-basics/explore-interactive/

## Step 1: アプリケーション設定の確認

前回デプロイしたアプリケーションが起動しているか確認しましょう。
`kubectl get`で存在しているPodを確認できます。

```
kubectl get pods
```

もしPodが起動していなければ、再度Podをリストしてください。

次は、Pod内のコンテナと使用されているイメージを確認するために`describe pods`コマンドを使います。

```
kubectl describe pods
```

ここでコンテナ一覧、IPアドレス、ポート番号、Podのライフサイクルのイベント一覧などのPodの詳細が確認できます。

describeコマンドの出力は広範囲に渡り、まだ説明していないコンセプトも含みます。
しかし心配しないでください。
このチュートリアルの最後にはそれらについても詳しくなっています。

## Step 2: ターミナル内でのアプリケーションの表示

前章での内容をおさらいしましょう。
標準では、すべてのPodはクラスタ内でのみアクセスできます。
ローカルマシンからアクセスするためにはローカルマシンとKubernetesクラスタ間のプロキシを作成する必要があります。

```
kubectl proxy
```

`proxy`コマンドはフォアグラウンドで起動します。
次からのコマンドは別のターミナルで実行してください。
新しいターミナルタブを開いてください。

Podの名前を取得し`POD_NAME`環境変数に格納します。

```
export POD_NAME=$(kubectl get pods -o go-template --template '{{range.items}}{{.metadata.name}}{{"\n"}}{{end}}')
echo Name of the Pod: $POD_NAME
```

アプリケーションの出力を得るには`curl`でリクエストします。

```
curl http://localhost:8001/api/v1/proxy/namespaces/default/pods/$POD_NAME/
```

## Step 3: コンテナログの取得

コンテナからログを得るには、`kubectl logs`コマンドを使います。

```
kubectl logs $POD_NAME
```

Podには1つのコンテナのみなので、コンテナ名を指定する必要はありません。

## Step 4: コンテナでのコマンド実行

コンテナ内で直接コマンドを実行することができます。
`exec`コマンドを使います。
環境変数の一覧を取得してみましょう。

```
kubectl exec $POD_NAME env
```

ログのときと同じように、Pod内にひとつしかコンテナがないのでコンテナの名前は省略できます。

次は、コンテナでbashセッションを開始しましょう。

```
kubectl exec -ti $POD_NAME bash
```

Node.jsアプリケーションを起動しているコンテナでコンソールを開けました。
ソースコードを表示してみましょう。

```
cat server.js
```

アプリケーションが起動しているかどうか`curl`で確認することもできます。

```
curl localhost:8080
```

閉じるには`exit`を入力します。
