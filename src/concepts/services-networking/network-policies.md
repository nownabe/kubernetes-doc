Network Policies
================

Network PolicyはPodのグループが他のネットワークエンドポイントとの通信をどのように許可されているかの仕様です。

`NetworkPolicy`リソースはPodの選択にラベルを使い、何のトラフィックが選択されたPodに対して許可されているかを定義します。


# Prerequisites

Network Policyはネットワークプラグインで実装されているので、`NetworkPolicy`をサポートするネットワーキングソリューションを使う必要があります。
そのコントローラなしにリソースを作成してもなんの効果もありません。


# Isolated and Non-isolated Pods

標準ではPodは隔離されておらず、あらゆるトラフィックを受け付けます。

特定のPodを選択するNetworkPolicyを持つことで、それらのPodは隔離されます。
あるNamespaceに一度でも特定のPodを選択するNetworkPolicyがあると、PodはNetworkPolicyにより許可されていない接続はすべて拒否します。
(Namespace外の選択されていないPodは引き続きすべてのトラフィックを許可します)
