Taints and Tolerations
======================

[Node affinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#node-affinity-beta-feature)はPodをあるNodeのセットに誘導するPodのプロパティです(優先度として、あるいはハードウェア要件)。
Taintは対象的にNodeがPodのセットを拒絶できるようになります。

TaintとTolerationは一緒に動作し、Podが不適切なNodeにスケジュールされないように保証します。
1つ以上のTaintが適用されたNodeは、そのTaintを許容してないPodは一切受け付けるべきではありません。
Tolerationが適用されたPodはTaintにマッチするNodeにもスケジュールされるようになります(必ずそのNodeにスケジュールされるわけではない)。

# Concepts

[`kubectl taint`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#taint)でTaintを追加できます。

```
kubectl taint nodes node1 key=value:NoSchedule
```

例えばこれはNode `node1`にTaintをセットしています。
Taintは`key`というキーと`value`という値を持ち、効果は`NoSchedule`です。
これはTolerationが一致しない限りこのNodeにスケジュールできるPodはないということを意味します。

このTaintを削除するには次のコマンドを実行します。

```
kubectl taint nodes node1 key:NoSchedule-
```

PodSpecでPodにTolerationを設定できます。
次のTolerationはどちらも上記のTaintにマッチします。
そして、いずれかのTolerationを設定されたPodは`node1`にスケジュールされる可能性があります。

```
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
```

```
tolerations:
- key: "key"
  operator: "Exists"
  effect: "NoSchedule"
```

`key`と`effect`が一致して、さらに次のいずれかの条件を満たせばTolerationとTaintはマッチします。

* `operator`が`Exists` (`value`は指定しなくていい)
* `operator`が`Equal`で`value`が等しい

`operator`はデフォルトでは`Equal`です。

次の2つのような特殊なケースもあります。

* 空の`key`かつ`operator`が`Exists`だとvalueとeffectは関係なくすべてのキーにマッチします

```
tolerations:
- operator: "Exists"
```

* 空の`effect`は`key`が一致するすべてにマッチします

```
tolerations:
- key: "key"
  operator: "Exists"
```

上記の例では`NoSchedule`という`effect`を使いました。
他に`PreferNoSchedule`を使うことができます。
これは`NoSchedule`のソフト版です。
システムは許容してないTaintのNodeを避けてPodを配置しようとしますが、必須ではありません。
3つ目の`effect`は`NoExecute`です。これは後で説明します。

1つのNodeに複数のTaintを設定したり、1つのPodに複数のTolerationを設定することができます。
Kubernetesはそれらをフィルターのように扱います。
すべてのNodeのTaintから始まり、PodのTolerationとマッチするものを無視します。
残ったTaintはそのPodに対して示された効果を発揮します。
特に、

* もしひとつでも`NoSchedule`のTaintが無視されず残れば、KubernetesはそのPodをそのNodeにスケジュールしません
* 無視されない`NoSchedule`がなく`PreferNoSchedule`のTaintがあれば、KubernetesはそのNodeにスケジュールしないように努力します
* ひとつでも無視されない`NoExecute`のTaintがあればPodはNodeから追い出されます(既にそのNode上で起動している場合)。そして、そのNode上にスケジュールされないようになります(まだNode上で起動していない場合)

例えば、次のように設定されたNodeを考えてください。

```
kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node1 key1=value1:NoExecute
kubectl taint nodes node1 key1=value2:NoSchedule
```

そして、Podは2つのTolerationを持ちます。

```
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
```

このケースでは、PodをそのNodeにスケジュールすることはできません。
なぜなら3つ目のTaintにマッチするTolerationがないからです。
しかし、すでにそのNode上で起動している場合はTaintが追加されても起動し続けることができます。
なぜなら3つ目のTaintがPodに許容されてない唯一のTaintだからです。

通常は、`NoExecute`のTaintがNodeに追加されたとき、そのTaintを許容していないすべてのPodはすぐに追い出され、許容しているPodは追い出されません。
しかし、`NoExecute`のTolerationは`tolerationSeconds`を指定することができ、Taintが追加されてからどれだけの期間そのNodeにいつづけるかを要求できます。
例えば、

```
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600
```

これは、このPodがマッチするTaintが追加されたNodeで起動済みの場合、Podは3600秒間そのノードにバインドされたままになり、その後追い出されます。
もしTaintがその時までに削除された場合、Podは追い出されません。
