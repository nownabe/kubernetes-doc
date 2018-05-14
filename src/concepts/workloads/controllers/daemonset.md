DaemonSet
=========

# What is a DaemonSet?

*DaemonSet*はすべて(またはいくつかの)NodeでPodのコピーを実行することを保証します。
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

# How Daemon Pods are Scheduled

# Communicating with Daemon Pods

# Updating a DaemonSet

# Alternatives to DaemonSet



