Horizontal Pod Autoscaler
=========================

Horizontal Pod AutoscalerはReplication Controller、Deployment、またはReplica SetのPod数を計測されたCPU使用率に基づいて自動でスケールします (他のアプリケーションが提供する[カスタムメトリック](https://git.k8s.io/community/contributors/design-proposals/instrumentation/custom-metrics-api.md)もサポートしています)。
DaemonSetなどのスケールできないオブジェクトに対して効果がないことに注意してください。

Horizontal Pod AutoscalerはKubernetesのAPIリソースとコントローラとして実装されています。
リソースはコントローラの振る舞いを決定します。
コントローラは計測される平均CPU使用率がユーザに指定された値にマッチするようにReplication ControllerやDeploymentのレプリカ数を定期的に調整します。

# How does the Horizontal Pod Autoscaler work?

![How does the Horizontal Pod Autoscaler work?](https://d33wubrfki0l68.cloudfront.net/4fe1ef7265a93f5f564bd3fbb0269ebd10b73b4e/1775d/images/docs/horizontal-pod-autoscaler.svg)

Horizontal Pod Autoscalerはコントローラマネージャの `--horizontal-pod-autoscaler-sync-period` フラグで指定される間隔によるコントロールループとして実装されています (デフォルト値は30秒)。

各ループではコントローラマネージャはHorizontalPodAutoscalerの定義にあるメトリクスの使用率をクエリします。
コントローラマネージャはリソースメトリクスAPI (各Podのリソースメトリクス) か、カスタムメトリクスAPI (すべての他のメトリクス)。

* 各Podのリソースメトリクス (CPUのような) は、HorizontalPodAutoscalerのターゲットPodそれぞれについてリソースメトリクスAPIからメトリクスを取得します。
そして、もし目標使用率が設定されていれば、コントローラは各Podのコンテナのリソースリクエストと同等の使用率を計算します。
もし目標値が設定されていれば、その値を直接使います。
コントローラはすべての対象Podの使用率または値 (指定されたターゲットの種類による) の平均をとり、正しいレプリカ数にスケールするために使われる比率を計算します。

もし、Podのいくつかのコンテナのリソースリクエストが設定されていなかった場合、PodのCPU使用量は定義されず、オートスケーラはそのメトリックに対してなにもしません。
後述する[algorithm details](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#algorithm-details)にオートスケーリングアルゴリズムがどのように動作するかがあるので見てください。

* 各Podのカスタムメトリクスは各Podのリソースメトリクスと同じように機能します。ただし、使用率は使えません。
* オブジェクトメトリクスと外部メトリクスは問題のオブジェクトを記述する単一のメトリックが取得されます。
メトリックは目標値と比較され、上述の比率が提供されます。
`autoscaling/v2beta2` APIバージョンでは比較される前に値をPod数で分割することもできます。

通常HorizontalPodAutoscalerは一連の集約されたAPIからメトリクスを取得します (`metrics.k8s.io`、`custom.metrics.k8s.io`、`external.metrics.k8s.io`)。
`metrics.k8s.io` APIは通常metrics-serverで提供され、別に起動される必要が有ります。
[metrics-server](https://kubernetes.io/docs/tasks/debug-application-cluster/core-metrics-pipeline/#metrics-server)のドキュメントを見てください。
HorizontalPodAutoscalerはHeapsterから直接メトリクスを取得することもできます。

> Heapsterからのメトリクス取得はKubernetes 1.11で非推奨になりました

詳しくは[Support for metrics APIs](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#support-for-metrics-apis)を見てください。

オートスケーラはScaleサブリソースを使って関連するスケーラブルコントローラ (Replication Controllers、Deployments、Replica Setsのような) にアクセスします。
Scaleはレプリカ数を動的に設定し、それらの状態を評価することを許可するためのインターフェイスです。
Scaleサブリソースについての詳細は[こちら](https://git.k8s.io/community/contributors/design-proposals/autoscaling/horizontal-pod-autoscaler.md#scale-subresource)で確認できます。
