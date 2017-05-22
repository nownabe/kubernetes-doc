アプリケーションの公開
======================

# Serviceを使ったアプリケーションの公開

https://kubernetes.io/docs/tutorials/kubernetes-basics/expose-intro/

## 目標
* Serviceについて学ぶ
* LabelとLabelSelectorがServiceとどのように関係しているかを理解する
* Serviceを使ってアプリケーションをKubernetesクラスタ外へ公開する

## Service概要

[Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)はダウンし得るものであり、実際に[ライフサイクル](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)を持ちます。
ワーカーNodeが死んだとき、同時にNodeで実行中のPodも失います。
そういった場合、[ReplicationController](https://kubernetes.io/docs/user-guide/replication-controller/#what-is-a-replicationcontroller)は動的にPodを新しく作成してアプリケーションの稼働を保つようにして、クラスタを期待された状態に保ちます。
他の例だと、画像処理バックエンドに3つのレプリカを検討してください。
これらのレプリカは代替可能です。
フロントエンドシステムはバックエンドのレプリカについて、Podが失われようと再作成されようと無関心でいられるべきです。
つまり、それぞれのPodは同じNodeにあるPodでさえKubernetesクラスタの中でユニークなIPを持ちますが、アプリケーションが機能し続けるためにはそれらが変わっても自動でアプリケーションと紐付ける方法が必要です。
ServiceはPodのセットとそれにアクセスするポリシーを表す論理的な概念です。
Serviceは依存Pod間の疎結合を可能にします。
Serviceは他のKubernetesオブジェクトと同じようにYAML([おすすめ](https://kubernetes.io/docs/concepts/configuration/overview/#general-config-tips))かJSONで定義されます。
Serviceの対象となるPodのセットは通常*LabelSelector*で決定されます。

PodはそれぞれユニークなIPアドレスを持ちますが、ServiceなしではそれらのIPはクラスタ外に公開されません。
Serviceはアプリケーションにトラフィックの受信を許します。
Serviceを公開する方法はいくつかあり、Specの中では`type`で指定されます。

* *ClusterIP* (default) - クラスタの内部IPとしてServiceを公開する。クラスタ内からのみアクセスできる。
* *NodePort* - NATで選択したNodeそれぞれの同じポートでServiceを公開する。クラスタ外から`:`を使ってアクセスできる。ClusterIPのスーパーセット。
* *LoadBalancer* - 対応しているクラウドであれば、クラウドのロードバランサを作成し、固定IPを割り当てる。NodePortのスーパーセット。
* *ExternalName* - CNAMEレコードによってServiceに`externalName`のような任意の名前をつける。プロキシを使わない。バージョン1.7以上の`kube-dns`が必要。

[Using Source IP](https://kubernetes.io/docs/tutorials/services/source-ip/)チュートリアルにより詳しい情報があります。
また、[Connecting Applications with Services](https://kubernetes.io/docs/concepts/services-networking/connect-applications-service)も見てください。

Serviceには`selector`の定義を伴わないユースケースもあることに注意してください。
`selector`を使わずServiceを作成すると、対応するエンドポイントオブジェクトは作成されません。
これは、開発者が手動でServiceと特定のエンドポイントをマッピングできます。
他に`selector`を作らない可能性としては、厳密に`ExternalName`を使っているということがあり得ます。

## ServiceとLabel

![Service](https://kubernetes.io/docs/tutorials/kubernetes-basics/public/images/module_04_services.svg)

ServiceはPod全体にトラフィックをルーティングします。
Serviceはアプリケーションに影響を与えずにPodに死と複製を許可する抽象概念です。
1つのアプリケーションのフロントエンドとバックエンドのような依存しているPod間の発見やルーティングはServiceが扱います。

ServiceはLabelとSelectorを使っているPodのセットと一致します。
これらはKubernetesのオブジェクトに対する論理的な操作を可能にします。
Labelはオブジェクトに紐付いたKey/Valueペアで、次のような使い方ができます。

* オブジェクトのdevelopment, test, productionの指示
* バージョンタグの埋め込み
* タグを使ったオブジェクトの分類

![Label and Selector](https://kubernetes.io/docs/tutorials/kubernetes-basics/public/images/module_04_labels.svg)

Labelは作成時かその後にオブジェクトに紐付けられます。
いつでも編集できます。

# インタラクティブチュートリアル

https://kubernetes.io/docs/tutorials/kubernetes-basics/expose-interactive/

## Step 1: Service作成

アプリケーションが起動しているか確認しましょう。
`kubectl get`コマンドでPodを確認してください。

```
kubectl get pods
```

次は現在のService一覧を表示しましょう。

```
kubectl get services
```

kubernetesというServiceがありますが、これはMinikubeが起動時に作成したものです。
新しいServiceを作って外部に公開するためにexposeコマンドでNodePortを設定します。

```
kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080
```

再度`get services`コマンドを実行しましょう。

```
kubectl get services
```

kubernetes-bootcampのServiceがあります。
ユニークなCluster-IPで受信していること、内部ポートと外部IP(NodeのIP)が確認できます。

NodePortによってどのポートが公開されたか確認するには`describe service`コマンドを使います。

```
kubectl describe service/kubernetes-bootcamp
```

NODE_PORT環境変数にNodeポートを格納してください。

```
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
echo NODE_PORT=$NODE_PORT
```

アプリケーションは公開されているので、`curl`でクラスタ外からアクセスできます。
NodeのIPと公開されたポートでアクセスしましょう。

```
curl host01:$NODE_PORT
```

## Step 2: Labelの使用

Deploymentは自動でPodにLabelを作っています。
`describe deployment`コマンドでLabelを確認できます。

```
kubectl describe deployment
```

Pod一覧表示のクエリにLabelを使ってみましょう。
`get pod`コマンドの`-l`オプションでLabelを指定します。

```
kubectl get pods -l run=kubernetes-bootcamp
```

Serviceでも同様にできます。

```
kubectl get services -l run=kubernetes-bootcamp
```

POD_NAME環境変数にPod名を格納しましょう。

```
export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
echo Name of the Pod: $POD_NAME
```

labelコマンドを使って新しいLabelを与えましょう。
labelコマンドはオブジェクトの種類、オブジェクトの名前を引数にとります。

```
kubectl label pod $POD_NAME app=v1
```

これはPodに新しいLabelを与えます(アプリケーションのバージョンをPodに付与)。
describe podコマンドで確認してみましょう。

```
kubectl describe pods $POD_NAME
```

Labelが付与されていることが確認できます。
またPod一覧のクエリで新しいLabelを使うこともできます。

```
kubectl get pods -l app=v1
```

## Step 3: Service削除

Serviceを削除するには`delete service`コマンドを使います。
ここでもLabelを使えます。

```
kubectl delete service -l run=kubernetes-bootcamp
```

削除されたか確認します。

```
kubectl get services
```

Serviceが削除されたことが確認できます。
ルーティングも削除されたかを`curl`で確認します。

```
curl host01:$NODE_PORT
```

これはもうクラスタ外からアプリケーションにアクセスできないことを示しています。
アプリケーションがPod内ではまだ起動していることを確認できます。

```
kubectl exec -ti $POD_NAME curl localhost:8080
```
