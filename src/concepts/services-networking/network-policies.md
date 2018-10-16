Network Policies
================

Network PolicyはPodのグループが他のネットワークエンドポイントとの通信をどのように許可されているかの仕様です。

`NetworkPolicy`リソースはPodの選択にラベルを使い、何のトラフィックが選択されたPodに対して許可されているかを定義します。


# Prerequisites

Network Policyはネットワークプラグインで実装されているので、`NetworkPolicy`をサポートするネットワーキングソリューションを使う必要があります。
そのコントローラなしにリソースを作成してもなんの効果もありません。

