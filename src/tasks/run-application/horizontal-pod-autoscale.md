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
コントローラはすべての対象Podの使用率または値 (指定されたターゲットの種類による) の平均をとり、期待するレプリカ数にスケールするために使われる比率を計算します。

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

## Algorithm Details

最も基本的な観点から、Horizontal Pod Autoscalerコントローラは期待するメトリック値と現在のメトリック値の比率に基づき操作します。

```
(期待レプリカ数) = cail[(現在のレプリカ数) * ( (現在のメトリック値) / (期待メトリック値) )]
```

例えば、現在のメトリック値が`200m`で、期待するメトリック値が`100m`とすると、`200.0 / 100.0 == 2.0`となるためレプリカ数は2倍されます。
逆に、現在が`50m`の場合は`50.0 / 100.0 == 0.5`となるためレプリカ数は半減されます。
もし比率が1.0に近い場合はスケーリングはスキップされます(`--horizontal-pod-autoscaler-tolerance`フラグでグローバルに設定される許容誤差です。デフォルトは0.1)。

`targetAverageValue`か`targetAverageUtilization`が指定されている時、`currentMetricValue`はHorizontalPodAutoscalerの対象と鳴っているすべてのPodの平均として計算されます。
どのような場合でも、許容誤差のチェックと最終的な値の決定の前に、Podのreadinessとメトリクスの紛失を考慮します。

削除タイムスタンプ付きのPod (すなわち、シャットダウンが始まったPod) とすべての失敗したPodは破棄されます。

もし特定のPodのメトリクスがない場合、あとで使うように避けられます。
メトリクスが欠落しているPodは最終的なスケーリング量を調整するために使われます。

CPUスケーリングのとき、もしいずれかのPodが準備中またはほとんどの最近のメトリックポイントが準備中のものだった場合、同じくどのPodは避けられます。

技術的制約のため、HorizontalPodAutoscalerコントローラは特定のCPUメトリクスを避けるかどうか決める時、Podが最初に準備完了になる時間を正確に測定できません。
そのかわりにもしPodがunreadyの場合は"not yet ready"とみなし、それが開始するまでの短く設定可能な時間幅でunreadyに移行します。
この値は`--horizontal-pod-autoscaler-initial-readiness-delay`フラグで設定できます。
デフォルト値は30秒です。
一度Podがreadyになれば、もしそれが長くて設定可能な時間までに開始したらreadyへのどんな移行も初回とみなします。
この値は`--horizontal-pod-autoscaler-cpu-initialization-period`フラグで設定できます。

`currentMetricValue / desiredMetricValue`の基本スケール比は避けられていないまたは上記から破棄されていないPodが計算に使われます。

もしメトリクスの欠損があれば、スケールダウンのときは期待値の100%、スケールアップのときは0%を消費すると仮定してより控えめに平均を再計算します。
これは潜在的なスケールの大きさを減少させます。

さらに、もしnot-yet-readyなPodがあり、欠損したメトリクスやnot-yet-ready Podを考慮せずスケールアップがあるとき、スケールアップの影響を小さくするためnon-yet-ready Podは0%を消費しているとみなします。

not-yet-ready Podと欠損メトリクスを考慮したあとで、使用比を再計算します。
もし新しい比でスケールの方向を反転させたり許容誤差内だった場合、スケーリングをスキップします。
さもなければ新しい比率をスケールに使用します。

もし複数のメトリクスがHorizontalPodAutoscalerで指定されていたら、それぞれのメトリックについて計算して、最大のレプリカ数を選択します。
もしいずれかのメトリクスが期待レプリカ数に変換できない場合(例えばメトリクスAPIからの取得エラー)、スケーリングはスキップされます。

最後に、HorizontalPodAutoscalerがターゲットをスケールする直前に、推奨スケールは保存されます。
コントローラは設定可能なウィンドウ内のすべての推奨を考慮しその中から最高の推奨を選びます。
この値は`--horizontal-pod-autoscaler-downscale-stabilization-window`フラグで設定でき、デフォルトは5分です。
これは、スケールダウンは徐々に行われ、メトリック値の急激な変化の影響を取り除きます。

# API Object

Horizontal Pod AutoscalerはKubernetesの`autoscaling` APIグループのAPIリソースです。
現在の安定バージョンの`autoscaling/v1`ではCPUによるオートスケーリングのみサポートしています。

ベータバージョンの`autoscaling/v2beta2`ではメモリとカスタムメトリクスによるスケーリングをサポートしています。
`autoscaling/v1`で動作するとき、`autoscaling/v2beta2`で導入された新しいフィールドはアノテーションとして保存されます。

APIオブジェクトの詳細は[HorizontalPodAutoscaler Object](https://git.k8s.io/community/contributors/design-proposals/autoscaling/horizontal-pod-autoscaler.md#horizontalpodautoscaler-object)を参照してください。

# Support for Horizontal Pod Autoscaler in kubectl

Horizontal Pod Autoscalerは他のAPIリソースのようにkubectlでサポートされています。
`kubectl create`コマンドで新しいautoscalerを作成できます。
`kubectl get hpa`で一覧表示でき、`kubectl describe hpa`で詳細を表示できます。
最後に、`kubectl delete hpa`で削除できます。

また、簡単にHorizontal Pod Autoscalerを作成するための特別な`kubectl autoscale`コマンドがあります。
例えば、`kubectl autoscale rs foo --min=2 --max=5 --cpu-percent=80`はレプリカセット`foo`に対するautoscalerを作成し、ターゲットCPU使用率を80%としてレプリカ数を2から5で調整します。
`kubectl autoscale`の詳細なドキュメントは[ここ](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#autoscale)にあります。

# Autoscaling during rolling update

現在のKubernetesでは、Replication Controllerを直接管理したりDeploymentオブジェクトを使って[ローリングアップデート](https://kubernetes.io/docs/tasks/run-application/rolling-update-replication-controller/)ができます。
Horizontal Pod Autoscalerは後者のみをサポートしています。
Horizontal Pod AutoscalerはDeploymentオブジェクトにバインドされ、Deploymentオブジェクトのサイズを設定し、DeploymentがReplica Setのサイズ設定に責任を持ちます。

Horizontal Pod AutoscalerはReplication Controllerで直接ローリングアップデートを操作してるときは働きません。すなわち、Horizontal Pod AutoscalerをReplication Controllerにバインドすることはできません(例えば`kubectl rolling-update`)。
その理由は、ローリングアップデートが新しいReplication Controllerを作る時、Horizontal Pod Autoscalerが新しいReplication Controllerをバインドできないからです。


# Support for cooldown/delay

Horizontal Pod Autoscalerを使ってレプリカグループのスケールを管理するとき、メトリクスの評価の動的な性質によりレプリカ数が頻繁に変動する可能性があります。
これはスラッシングとも言われます。

v1.6から、クラスタオペレータは`kube-controller-manager`のフラグとしてグローバルなHorizontal Pod Autoscalerの設定をチューニングすることでこの問題を緩和できます。

v1.12から新しいアルゴリズムの更新により遅延は必要なくなりました。

* `--horizontal-pod-autoscaler-downscale-delay`: このオプションの値は、現在のダウンスケールが終わったあと、他のダウンスケールが実行前にどれだけ待たないといけないかを指定します。
デフォルト値は5分です。

> Note: これらのパラメータをチューニングするとき、クラスタオペレータは発生しうる結果に注意してください。
> もし遅延(クールダウン)値が非常に長く設定されていたら、Horizontal Pod Autoscalerがワークロードの変化に反応しないということになります。
> 短く設定された場合、よくスラッシングが発生してしまいます。


# Support for multiple metrics

Kubernetes 1.6で複数のメトリクスによるスケーリングがサポートされました。
`autoscaling/v2beta2` APIでHorizontal Pod Autoscaleに複数のメトリクスを指定できます。
そして、Horizontal Pod Autoscalerコントローラはそれぞれのメトリックを評価し、それに基づきスケールを提案します。
最も大きいスケールが適用されます。


# Support for custom metrics

Kubernetes 1.2で特殊なアノテーションを用いたアプリケーションメトリクスによるスケーリングがアルファサポートされました。
Kubernetes 1.6で新しいAPIのためアノテーションは廃止されました。
従来のカスタムメトリクス収集方法は引き続き利用できますが、それらのメトリクスはHorizontal Pod Autoscalerでは使えません。
また、スケールのカスタムメトリクスを指定するアノテーションはもうHorizontal Pod Autoscalerコントローラでは使えません。


# Support for metrics APIs

標準で、HorizontalPodAutoscalerコントローラは一連のAPIからメトリクスを取得します。
APIにアクセスするために、クラスタ管理者は次のことを確認してください。

* [API aggregation layer](https://kubernetes.io/docs/tasks/access-kubernetes-api/configure-aggregation-layer/)が有効
* 対応するAPIが登録されている
  * リソースメトリクスのための`metrics.k8s.io` API。
    一般的に[metrics-server](https://github.com/kubernetes-incubator/metrics-server)によって提供
    クラスタアドオンとして起動できます。
  * カスタムメトリクスのための`custom.metrics.k8s.io` API。
    メトリクスベンダーの"adapter" APIサーバで提供されます。
  * 外部メトリクスのための`external.metrics.k8s.io` API。
    これはおそらく上記のカスタムメトリクスアダプタによって提供されます。
* `--horizontal-pod-autoscaler-use-rest-clients`が`true`またはセットされていない。
  これが`false`だとHeapsterがオートスケーリングに使われますが非推奨です。
