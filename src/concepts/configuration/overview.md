Configurationベストプラクティス
===============================

このドキュメントはuser-guideやgetting-startedのドキュメントや例で導入されたConfigurationのベストプラクティスを一箇所にまとめたものです。
これは生きているドキュメントなので、もし便利だけどここにないトピックなどがあればIssueやPRを送ってください。

# 一般的なTips
* Configurationの定義には、最新の安定APIバージョンを指定する(現在はv1)。
* Configurationファイルはクラスタに送信される前にバージョン管理されるべき。これで必要ならすぐにロールバックすることができます。また、再作成や復元も助けます。
* JSONよりYAMLで書きましょう。ほとんどの場合でお互い変換できますが、YAMLの方が人に親切です。
* 関係するオブジェクトを一緒にひとつのファイルにまとめるのは意味があります。この方法は分割されたファイルより管理が簡単です。例として[guestbook-all-in-one.yaml](https://github.com/kubernetes/kubernetes/tree/master/examples/guestbook/all-in-one/guestbook-all-in-one.yaml)を見てください。(Note also that many kubectl commands can be called on a directory, and so you can also call kubectl create on a directory of config files— see below for more detail)。
* 不必要なデフォルト値は設定しないでください。設定をシンプル、ミニマムにしてエラーを減らすためです。For example, omit the selector and labels in a `ReplicationController` if you want them to be the same as the labels in its `podTemplate`, since those fields are populated from the `podTemplate` labels by default. See the [guestbook app’s](https://github.com/kubernetes/kubernetes/tree/master/examples/guestbook/) .yaml files for some [examples](https://github.com/kubernetes/kubernetes/tree/master/examples/guestbook/frontend-deployment.yaml) of this.
* Put an object description in an annotation to allow better introspection.

# 生Pod VS Replication ControllerとJob

# Service

# Label

# コンテナイメージ

# kubectl
