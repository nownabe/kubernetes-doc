Cluster作成
===========

# Minikubeを使ってClusterを作成しよう

https://kubernetes.io/docs/tutorials/kubernetes-basics/cluster-intro/

## 目標
* Kubernetes clusterが何かを学ぶ
* Minikubeが何かを学ぶ
* オンラインターミナルでKubernetes clusterを起動する

## Kubernetes Clusters

**Kubernetesはひとつの単位として動作するために接続されたコンピュータ群を高可用なクラスタとして調整します。**
Kubernetesの抽象化を利用すれば個々のマシンに縛られずコンテナ化されたアプリケーションをクラスタにデプロイできます。
この新しいデプロイモデルを使うためには、アプリケーションは個々のホストから分離された方法でパッケージされる必要があります。
つまり、コンテナ化です。
深くホストと結合したパッケージとして特定のマシンに直接インストールされることに比べ、コンテナ化アプリケーションはより柔軟で可用性があります。
**Kubernetesはクラスタ全体のアプリケーションコンテナの配布とスケジューリングをより効率的に自動化します。**

## Cluster Diagram
![Kubernetes Cluster](https://kubernetes.io/docs/tutorials/kubernetes-basics/public/images/module_01_cluster.svg)

**Masterはクラスタを管理します。**
Masterは次のようなクラスタのすべてのアクティビティを調整します。

* アプリケーションのスケジューリング
* アプリケーションの状態のメンテナンス
* アプリケーションのスケーリング
* ロールアウトアップデート

**NodeはKubernetesクラスタでワーカーマシンとして動作する仮想マシンか物理マシンです。**
それぞれのNodeはノード管理とMasterとの通信を行うKubeletというエージェントを持ちます。
また、Nodeは[Docker](https://www.docker.com/)や[rkt](https://coreos.com/rkt/)のようなコンテナ操作のツールをもつべきです。
本番のトラフィックをKubernetesクラスタで扱うためには最低3台のNodeが必要です。

Kubernetesにアプリケーションをデプロイするとき、開発者はMasterにアプリケーションコンテナ起動に指示を出します。
MasterはクラスタのNodeでのアプリケーションの起動をスケジュールします。
**ノードはKubernetes APIを使ってMasterと通信します。**
エンドユーザもクラスタとやり取りするために直接Kubernetes APIを使うことができます。

Kubernetesクラスタは物理マシンか仮想マシンどちらでもデプロイできます。
Kubernetesのデプロイを体験するために[Minikube](https://github.com/kubernetes/minikube)が使えます。
Minikubeは軽量なKubernetes実装で、ローカル上の仮想マシンにシンプルなクラスタをデプロイします。
MinikubeはLinux、Mac OS、Windowsで利用可能です。
Minikube CLIはクラスタの起動、停止、状態確認、削除などの基本的な操作を提供します。
このチュートリアルでは、あなたは準備済みのMinikubeをオンラインターミナルで使用することになります。

Kubernetesが何者かは知りました。次はオンラインチュートリアルで最初のクラスタを起動しましょう！

# インタラクティブチュートリアル
https://kubernetes.io/docs/tutorials/kubernetes-basics/cluster-interactive/
