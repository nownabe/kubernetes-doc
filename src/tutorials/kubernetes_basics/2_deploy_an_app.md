アプリケーションデプロイ
========================

# kubectlを使ったDeploymentの作成

https://kubernetes.io/docs/tutorials/kubernetes-basics/deploy-intro/

## 目標

* アプリケーションのDeploymentについて学ぶ
* kubectlを使ってKubernetesに最初のアプリケーションをデプロイする

## Kubernetes Deployments

前回Kubernetesクラスタを起動したので、その上にコンテナ化されたアプリケーションをデプロイすることができます。
それには、Kubermentesの**Deployment**を作る必要があります。
Deploymentはアプリケーションインスタンスの作成と更新の責務を担います。
一度Deploymentを作れば、KubernetesのMasterはDeploymentによって作られたアプリケーションインスタンスをクラスタの個々のノード上にスケジュールします。

一度アプリケーションインスタンスが作られたら、Kubernetes Deployment Controllerはインスタンスを監視し続けます。
もしインスタンスをもつNodeが落ちたり削除されたりしたら、Deploymentはインスタンスを移動します。
**これはマシンの障害やメンテナンスを扱うための自己回復メカニズムを提供します。**

プリオーケストレーションの世界ではアプリケーションの開始にインストールスクリプトが使われるでしょうが、それらのスクリプトはマシン障害からの復旧は行いません。
アプリケーションインスタンスの作成とNodeで稼働状態を保つことにより、Kubernetes Deploymentはアプリケーション管理について根本から異なるアプローチを提供します。

## Kubernetesへのデプロイ

![Deployment](https://kubernetes.io/docs/tutorials/kubernetes-basics/public/images/module_02_first_app.svg)

KubernetesのCLIである**kubectl**を使うことでDeploymentの作成と管理ができます。
kubectlはクラスタと通信するためにKubernetes APIを使います。
この章ではKubernetesクラスタでアプリケーションを動作させるためのDeploymentの作成に必要なkubectlのコマンドを学びます。

Deploymentを作成するとき、アプリケーションのコンテナイメージと起動したいreplica数を指定する必要があります。
これらの情報はあとでDeploymentを更新して変更することができます。
5、6章でDeploymentのスケーリングや更新の方法を議論します。

私たちの最初のDeploymentでは、Node.jsアプリケーションのDockerコンテナを使います。
ソースコードとDockerfileは[GitHub repository](https://github.com/kubernetes/kubernetes-bootcamp)にあります。

Deploymentについて学びました。それではオンラインチュートリアルで最初のアプリケーションをデプロイしましょう！

# インタラクティブチュートリアル
https://kubernetes.io/docs/tutorials/kubernetes-basics/deploy-interactive/
