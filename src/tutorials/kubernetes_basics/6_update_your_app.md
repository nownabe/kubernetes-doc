アプリケーションの更新
======================

# ローリングアップデート

https://kubernetes.io/docs/tutorials/kubernetes-basics/update-intro/

## 目標
* kubectlを使ってローリングアップデートする

## アプリケーションの更新
ユーザはアプリケーションに無停止を期待していて、開発者は一日に何回ものデプロイすることを期待されています。
Kubernetesではローリングアップによってこれを成し遂げます。
**ローリングアップデート**はインクリメンタルにPodのインスタンスを新しいものに更新することでダウンタイムなしのアップデートを実現します。
新しいPodは利用可能なりソースがあるNodeにスケジュールされます。

前章では複数インスタンスを起動してアプリケーションをスケールしました。
これはアプリケーションの可用性を損なわずにアップデートするために必要です。
標準では、アップデート時に利用不可能になり得るPod数も作成可能な新しいPod数も1です。
これらの設定はPodについて具体数か割合で指定することができます。
Kubernetesでは、更新はバージョン管理されていて、どのDeploymentの更新も前のバージョンに戻すことが可能です。

## ローリングアップデート概要

![1](https://kubernetes.io/docs/tutorials/kubernetes-basics/public/images/module_06_rollingupdates1.svg)
![2](https://kubernetes.io/docs/tutorials/kubernetes-basics/public/images/module_06_rollingupdates2.svg)
![3](https://kubernetes.io/docs/tutorials/kubernetes-basics/public/images/module_06_rollingupdates3.svg)
![4](https://kubernetes.io/docs/tutorials/kubernetes-basics/public/images/module_06_rollingupdates4.svg)

アプリケーションのスケーリングと同じように、Deploymentが外部に公開されている場合はServiceはアップデート中、利用可能なPodにのみトラフィックを分散します。
利用可能なPodとは、アプリケーションのユーザにとって利用可能ということです。

ローリングアップデートは次のアクションを許可します。

* Promote an application from one environment to another (via container image updates)
* 前のバージョンへのロールバック
* ゼロダウンタイムでのアプリケーションの継続的インテグレーションと継続的デプロイ

インタラクティブチュートリアルではアプリケーションの新しいバージョンへのアップデートとロールバックを行います。

# インタラクティブチュートリアル

https://kubernetes.io/docs/tutorials/kubernetes-basics/update-interactive/

## Step 1: アプリケーションのバージョンの更新

`get deployments`コマンドでDeploymentの一覧を表示しましょう。

```
kubectl get deployments
```

`get pods`コマンドで起動中のPodの一覧を表示してください。

```
kubectl get pods
```

現在のアプリケーションのイメージバージョンを見るためにPodに対して`describe`コマンドを実行してください。

```
kubectl describe pods
```

アプリケーションのイメージをバージョン2にアップデートするために`set image`コマンドを実行しましょう。
Deployment名と新しいバージョンのイメージを引数にとります。

```
kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2
```

コマンドはDeploymentが別のイメージを使っていることを知らせて、ローリングアップデートを開始します。
`get pods`コマンドで新しいPodの状態と終了中の古いPodを確認できます。

```
kubectl get pods
```

## Step 2: アップデートの確認

まずはアプリケーションが起動しているか確認しましょう。
`describe service`で公開されているIPとポートを確認します。

```
kubectl describe services/kubernetes-bootcamp
```

NODE_PORT環境変数にNodeポートを格納します。

```
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
echo NODE_PORT=$NODE_PORT
```

次に`curl`で確認します。

```
curl host01:$NODE_PORT
```

毎回異なるPodにリクエストして、最新バージョン(v2)のPodが起動していることがわかります。

rollout statusコマンドでアップデートについて確認することもできます。

```
kubectl rollout status deployments/kubernetes-bootcamp
```

再度describeコマンドで現在のイメージバージョンを確認しましょう。

```
kubectl describe pods
```

## Step 3: ロールバック

v10にアップデートしてみましょう。

```
kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v10
```

`get deployments`で状態を確認してください。

```
kubectl get deployments
```

何かおかしいですね。期待される数のPodが利用可能になっていません。
Pod一覧を表示してみます。

`describe`コマンドで詳しくみてみます。

```
kubectl describe pods
```

v10がリポジトリに存在していません。
以前の正常なバージョンにロールバックしましょう。
`rollout undo`コマンドを使います。

```
kubectl rollout undo deployments/kubernetes-bootcamp
```

rolloutコマンドは以前の状態を知っていて(v2イメージ)、deploymentをその状態に戻します。
アップデートはバージョン管理されていて、それ以前のどの状態にもDeploymentを戻すことができます。
再びPod一覧を表示しましょう。

```
kubectl get pods
```

4つのPodがRunningになっています。
再度イメージを確認しましょう。

```
kubectl describe pods
```

安定バージョンであるv2が使われていて、ロールバックが成功していることがわかります。
