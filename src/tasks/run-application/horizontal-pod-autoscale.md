Horizontal Pod Autoscaler
=========================

Horizontal Pod AutoscalerはReplication Controller、Deployment、またはReplica SetのPod数を計測されたCPU使用率に基づいて自動でスケールします (他のアプリケーションが提供する[カスタムメトリック](https://git.k8s.io/community/contributors/design-proposals/instrumentation/custom-metrics-api.md)もサポートしています)。
DaemonSetなどのスケールできないオブジェクトに対して効果がないことに注意してください。

Horizontal Pod AutoscalerはKubernetesのAPIリソースとコントローラとして実装されています。
リソースはコントローラの振る舞いを決定します。
コントローラは計測される平均CPU使用率がユーザに指定された値にマッチするようにReplication ControllerやDeploymentのレプリカ数を定期的に調整します。
