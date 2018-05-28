Using RBAC Authorization
========================

Role-Based Access Control ("RBAC") は認可を得るために "rbac.authorization.k8s.io" APIグループを使い、Kubernetes APIに対する動的なポリシー管理を提供します。

1.8ではRBACはstableであり、rbac.authorization.k8s.io/v1 APIで提供されます。

RBACを有効化するにはapiserverを`--authorization-mode=RBAC`で起動します。

# API Overview

RBAC APIはこのセクションの4タイプを定義します。
ユーザは他のAPIリソースと同じようにこれらを扱えます(`kubectl`やAPIコールなど)。
例えば、`kubectl create -f (resource).yml`はこれらの例のいずれにも使えますが、理解するためにはまずこのセクションをみるべきです。

## Role and ClusterRole

RBAC APIでは1つのRoleはPermissionの集合を表すルールを持ちます。
Permissionは純粋に許可するものです(拒否のルールはありません)。
RoleはNamespace内では`Role`として、クラスター全体には`ClusterRole`として定義できます。

`Role`は1つのNamespace内でアクセスを与えるものとしてのみ使えます。
ここに、"default" namespace内でPodへの読み取り権限を与える`Role`の例があります。

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

`ClusterRole`は`Role`と同じく権限を与えますが、クラスタをスコープとします。

* クラスタスコープのリソースのため (Nodeのような)
* リソースでないエンドポイント ("/healthz"のような)
* すべてのNamespaceにおける、Namespaceスコープのリソース (Podのような)。例えば、`kubectl get pods --all-namespaces`

次の`ClusterRole`は特定のNamespaceまたはすべての名前空間のSecretへの読み取り権限を与えます(どうBindされるかによります)。

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  # "namespace" omitted since ClusterRoles are not namespaced
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

## RoleBinding and ClusterRoleBinding

RoleBindingは1つのRoleに定義されたPermissionをユーザまたはユーザの集合に付与します。
RoleBindingはSubject (User、Group又はService Account) のリストと付与されるRoleへの参照を持ちます。
Permissionは`RoleBinding`によってNamespace内に、`ClusterRoleBinding`によってクラスタ全体に対して付与できます。

`RoleBinding`は同じNamespaceの`Role`を参照するでしょう。
次の`RoleBinding`はNamespace "default"内においてRole "pod-reader"をUser "jane"に付与します。
これは、Namespace "default"内で"jane"がPodを閲覧することを許可します。

`roleRef`は実際にbindingを作る方法です。
`kind`は`Role`か`ClusterRole`のどちらかで、`name`は指定する`Role`か`ClusterRole`を参照します。
下の例でこのRoleBindingは`roleRef`でUser "jane"を上で作った`pod-reader`という`Role`にバインドしています。

```
# This role binding allows "jane" to read pods in the "default" namespace.
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

1つの`RoleBinding`は`RoleBinding`のNamespace内において、Namespaceスコープのリソースに対して`ClusterRole`で定義されたPermissionを付与するために`ClusterRole`を参照することがあります。
これによりクラスタ全体で共通のRoleを定義して複数のNamespaceにおいて再利用できます。

例えば、次の`RoleBinding`は`ClusterRole`を参照していますが、"dave" (対象のSubjectで大文字小文字は区別される) は"development" Namespace (`RoleBinding`のNamespace) のSecretのみ閲覧できます。

```
# This role binding allows "dave" to read secrets in the "development" namespace.
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-secrets
  namespace: development # This only grants permissions within the "development" namespace.
subjects:
- kind: User
  name: dave # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

最後に、`ClusterRoleBinding`はクラスタレベルとすべてのNamespaceに対してPermissionを付与します。
次の`ClusterRoleBinding`は"manager" Groupのすべてのユーザに対してどのNamespaceでもSecretの閲覧を許可します。

```
# This cluster role binding allows anyone in the "manager" group to read secrets in any namespace.
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

## Referring to Resources

ほとんどのリソースは"pods"のような文字列で名前が表され、関係するAPIのエンドポイントのURLとして現れます。
しかしいくつかのKubernetes APIはPodのログのような"subresource"に関係します。
PodのログのエンドポイントのURLは

```
GET /api/v1/namespaces/{namespace}/pods/{name}/log
```

このケースでは、"pods"がNamespaceリソース、"log"がPodのサブリソースです。
これをRBAC Roleで表すにはリソースとサブリソースをスラッシュで区切ります。
subjectにPodとPodのログの両方に読み取り権限を与えるには、次のようにします。

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-and-pod-logs-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list"]
```

リソースは`resourceNames`リストを介して特定のリクエストのために名前で指定することができます。
指定した場合は、個々のリソースに対して"get"、"delete"、"update"、"patch"の動詞を使ってリクエストを制限できます。
あるSubjectのあるひとつのConfigMapに対する操作を"get"と"update"のみに制限する場合は、このようになります。

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: configmap-updater
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["my-configmap"]
  verbs: ["update", "get"]
```

特に、もし`resourceNames`がセットされていると、動詞はlist、watch、createまたはdeletecollectionではいけない。
なぜなら、create、list、watch、deletecollectionのAPIリクエストには名前は現れないからです。
これらの動詞は`resourceNames`がセットされたルールは許可されない。
ルールの`resourceNames`の部分がリクエストとマッチしないからです。

## Aggregated ClusterRoles

1.9では`aggregationRule`を使うと他のClusterRoleを結合してClusterRoleを作成できます。
aggregateされたClursterRoleのpermissionはコントローラに管理され、提供されたlabelセレクタにマッチするすべてのClusterRoleと統合する形で埋められます。
aggregateされたClusterRoleの例は、

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: [] # Rules are automatically filled in by the controller manager.
```

labelセレクタにマッチするClusterRoleの作成はaggregateされたClusterRoleにルールを追加します。
このケースでは`rbac.example.com/aggregate-to-monitoring: true`のラベルを持つ他のClusterRoleを作成することで、"monitoring" ClusterRoleにルールが追加されます。

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: monitoring-endpoints
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
# These rules will be added to the "monitoring" role.
rules:
- apiGroups: [""]
  resources: ["services", "endpoints", "pods"]
    verbs: ["get", "list", "watch"]
```

後述する標準でユーザが触れるRoleはClusterRole aggregationを使っています。
これにより管理者はデフォルトRole上でCustomeResourceDefinisionsやAggregated APIサーバにより提供されるようなカスタムリソースのためのルールを含めることができます。

例えば、次のClusterRoleはデフォルトRoleである"admin"と"edit"にカスタムリソース"CronTabs"を管理させ、"view" Roleにこのリソースへの読み取りをさせます。

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: aggregate-cron-tabs-edit
  labels:
    # Add these permissions to the "admin" and "edit" default roles.
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
rules:
- apiGroups: ["stable.example.com"]
  resources: ["crontabs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: aggregate-cron-tabs-view
  labels:
    # Add these permissions to the "view" default role.
    rbac.authorization.k8s.io/aggregate-to-view: "true"
rules:
- apiGroups: ["stable.example.com"]
  resources: ["crontabs"]
  verbs: ["get", "list", "watch"]
```

### Role Examples

`rules`セクションの例を示します。

core APIグループの"pods"リソースへの読み取りを許可します。

```
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

"extensions"と"apps" APIグループの"deployments"の読み書きを許可します。

```
rules:
- apiGroups: ["extensions", "apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

"pods"の読み取りと"jobs"の読み書きを許可します。

```
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch", "extensions"]
  resources: ["jobs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

`ConfigMap`の"my-config"の読み取りを許可します。(`RoleBinding`でひとつのNamespaceのひとつの`ConfigMap`に制限する必要があります)

```
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["my-config"]
  verbs: ["get"]
```

coreグループの"nodes"リソースの読み取りを許可します。(`Node`はクラスタスコープなので、`ClusterRole`で定義され`ClusterRoleBinding`でバインドされる必要がります)

```
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

リソースでない"/healthz"エンドポイントとサブパスへの"GET"と"POST"リクエストを許可します。

```
rules:
- nonResourceURLs: ["/healthz", "/healthz/*"] # '*' in a nonResourceURL is a suffix glob match
  verbs: ["get", "post"]
```

### Role Binding Examples

## Referring to Subjects

`RoleBinding`又は`ClusterRoleBinding`はRoleをSubjectにバインドします。
Subjectはグループ、ユーザー、ServiceAccountです。

ユーザーは文字列で表されます。
"alice"のようなプレーンユーザー名、"bob@example.com"のようなemailスタイル、文字列で数値IDが使えます。
[authentication modules](https://kubernetes.io/docs/admin/authentication/)を設定して希望のフォーマットでユーザ名を生成するのはKubernetes管理者に委ねられます。
RBAC認証システムは特定のフォーマットであることを要求しません。
しかし、`system:`プリフィックスはKubernetesシステムに予約されています。
管理者はユーザー名にこのプリフィックスを含めないようにする必要があります。

現在はKubernetes内のグループ情報はAuthenticatorモジュールによって提供されます。
グループはユーザーのように文字列で表されます。
`system:`の予約プリフィックスを除き、グループは特定のフォーマットはありません。

[Service Account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)は`system:serviceaccount:`というプレフィックスのユーザー名を持ち、`system:serviceaccounts:`というプレフィックスのグループに属します。

### Role Binding Examples

`RoleBinding`の`subjects`セクションの例を示します。

"alice@example.com" ユーザー:

```
subjects:
- kind: User
  name: "alice@example.com"
  apiGroup: rbac.authorization.k8s.io
```

"frontend-admins" グループ:

```
subjects:
- kind: Group
  name: "frontend-admins"
  apiGroup: rbac.authorization.k8s.io
```

kube-system NamespaceのデフォルトのService Account:

```
subjects:
- kind: ServiceAccount
  name: default
  namespace: kube-system
```

"qa" NamespaceのすべてのService Account:

```
subjects:
- kind: Group
  name: system:serviceaccounts:qa
  apiGroup: rbac.authorization.k8s.io
```

すべてのNamespaceのすべてのService Account:

```
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io
```

すべての認証済みユーザ (バージョン1.5以上):

```
subjects:
- kind: Group
  name: system:authenticated
  apiGroup: rbac.authorization.k8s.io
```

すべての認証していないユーザ (バージョン1.5以上):

```
subjects:
- kind: Group
  name: system:unauthenticated
  apiGroup: rbac.authorization.k8s.io
```

すべてのユーザ (バージョン1.5以上):

```
subjects:
- kind: Group
  name: system:authenticated
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: system:unauthenticated
  apiGroup: rbac.authorization.k8s.io
```

# Default Roles and Role Bindings

APIサーバはデフォルトの`ClusterRole`と`ClusterRoleBinding`オブジェクトを生成します。
それらの多くは`system:`というプレフィックスが付き、そのリソースがインフラに所有されていることを示します。
それらのリソースの変更はクラスタの破壊を引き起こす可能性があります。
例えば`system:node`はClusterRoleです。
このClusterRoleはkubeletesの権限を定義しています。
もしこれを変更するとkubeletが正常に動作しなくなります。

クラスタのすべてのデフォルトroleとrolebindingは`kubernetes.io/bootstrapping=rbac-defaults`というラベルがついています。

## Auto-reconciliation

起動時には毎回APIサーバはデフォルトのClusterRoleを足りない権限が内容に更新し、デフォルトのClusterRoleBindingを足りないSubjectがないように更新します。
これは予期しない変更から回復し、新しいリリースでの変更に追随します。

このreconciliationの対象から外れるには、デフォルトのClusterRoleの`rbac.authorization.kubernetes.io/autoupdate`アノテーションを`false`にセットします。
デフォルトの権限やsubjectが足りないとクラスタが動作しない可能性があるので注意してください。

Auto-reconciliationはKubernetes 1.6以上でRBACが有効のとき有効になります。

## Discovery Roles

| Default ClusterRole | Default ClusterRoleBinding | Description |
|---|---|---|
| **system:basic-user** | **system:authenticated**と**system:unauthenticated**グループ | 彼らに関する基本的な情報の読み取りを許可 |
| **system:discovery** | **system:authenticated**と**system:unauthenticated**グループ | APIレベルで交渉し発見するのに必要なAPI discoveryエンドポイントへの読み取りを許可 |

## User-fasing Roles

いくつかのデフォルトRoleは`system:`プリフィックスがついていません。
これらはユーザが直面するよう意図されています。
`cluster-admin`はスーパーユーザーのroleであり、ClusterRoleBinding (`cluster-status`)`でクラスタ全体に対して付与されます。
特定のNamespaceのための`admin`、`edit`、`view`はRoleBindingで付与されます。

1.9からuser-facing roleはカスタムリソースへのルールを管理者に付与できるように[ClusterRole Aggregation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#aggregated-clusterroles)を使っています。
ルールを"admin"、"edit"、"view"に追加するには、以下のラベルを一つ以上持ったClusterRoleを作成します。

```
metadata:
  labels:
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-view: "true"
```

| Default ClusterRole | Default ClusterRoleBinding | Description |
|---|---|---|
| **cluster-admin** | **system:masters** グループ | いかなるリソースへのいかなるアクションも可能なスーパーユーザーアクセスを許可します。**ClusterRoleBinding**でバインドされたとき、クラスタ内、全Namespace内のすべてのリソースに対するフルコントロール権限を与えます。**RoleBinding**でバインドされたとき、Namespace自身とそのNamespace内でのすべてのリソースに対するフルコントロール権限を与えます。 |
| **admin** | None | 管理アクセスを許可します。**RoleBinding**で特定のNamespaceにバインドされることを想定しています。**RoleBinding**でバインドされたとき、Namespace内のほぼすべてのリソースに対する読み書き権限が与えられます。RoleとRoleBindingの作成もできます。Namespace自身とリソースクオータに対する書き込み権限はありません。 |
| **edit** | None | Namespace内のほぼすべてのオブジェクトに対する読み書きを許可します。RoleとRoleBindingに対する読み書きはできません。 |
| **view** | None | Namespace内のほぼすべてのオブジェクトに対する読み取りを許可します。RoleとRoleBindingに対する読み取りはできません。エスカレートされているSecretは閲覧できません。 |

## Core Component Roles

| Default ClusterRole | Default ClusterRoleBinding | Description |
|---|---|---|
| **system:kube-scheduler** | **system:kube-scheduler**ユーザ | kube-schedulerコンポーネントが必要とするリソースへのアクセスを許可します。 |
| **system:kube-controller-manager** | **system:kube-controller-manager**ユーザ | kube-controller-managerが必要とするリソースへのアクセスを許可します。個々のコントロールループが必要とする権限は[controller roles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#controller-roles)が含んでいます。 |
| **system:node** | 1.8以上はなし | kubeletが必要とするリソースへのアクセスを許可します。すべてのSecretへの読み取りとすべてのPodステータスオブジェクトへの書き込み権限を含みます。1.7からはこのRoleではなく[Node authorizer](https://kubernetes.io/docs/admin/authorization/node/)と[NodeRestriction admission plugin](https://kubernetes.io/docs/admin/admission-controllers/#noderestriction)が推奨されています。 (省略) 1.8以上では自動ではbindingされません。 |
| **system:node-proxier** | **system:kube-proxy**ユーザ | kube-proxyが必要とするリソースへのアクセスを許可します。 |

## Other Component Roles

| Default ClusterRole | Default ClusterRoleBinding | Description |
|---|---|---|
| **system:auth-delegator** | None | 委譲された認証・認可チェックを許可します。一般には統合された認証・認可のためのアドオンのAPIサーバーに使われます。 |
| **system:heapster** | None | [Heapster](https://github.com/kubernetes/heapster)用のRoleです。 |
| **system:kube-aggregator** | None | [kube-aggregator](https://github.com/kubernetes/kube-aggregator)用のRoleです。 |
| **system:kube-dns** | **kube-system** Namespaceの**kube-dns** ServiceAccount | [kube-dns](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)用のRoleです。 |
| **system:kubelet-api-admin** | None | kubelet APIへのフルアクセス |
| **system:node-bootstrapper** | None | [Kubelet TLS bootstrapping](https://kubernetes.io/docs/admin/kubelet-tls-bootstrapping/)の動作に必要なリソースへのアクセスを許可します。 |
| **system:node-problem-detector** | None | [node-problem-detector](https://github.com/kubernetes/node-problem-detector)用のRoleです。|
| **system:persistent-volume-provisioner** | None | 多くの[dynamic volume provisioners](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#provisioner)が必要とするリソースへのアクセスを許可します。 |

## Controller Roles

[Kubernetes controller manager](https://kubernetes.io/docs/admin/kube-controller-manager/)は核となるコントロールループを実行します。
`--use-service-account-credentials`で実行されたとき、各コントロールループは分離されたServiceAccountを使用して開始されます。
`system:controller:`プレフィックスではじまるRoleがそれぞれのコントロールループのために存在します。
controller managerが`--use-service-account-credentials`を伴わす開始された場合は、すべてのコントロールループは自身の認証情報を使います。
これは関連するRoleがすべて与えられていなければいけません。

* system:controller:attachdetach-controller
* system:controller:certificate-controller
* system:controller:cronjob-controller
* system:controller:daemon-set-controller
* system:controller:deployment-controller
* system:controller:disruption-controller
* system:controller:endpoint-controller
* system:controller:generic-garbage-collector
* system:controller:horizontal-pod-autoscaler
* system:controller:job-controller
* system:controller:namespace-controller
* system:controller:node-controller
* system:controller:persistent-volume-binder
* system:controller:pod-garbage-collector
* system:controller:pv-protection-controller
* system:controller:pvc-protection-controller
* system:controller:replicaset-controller
* system:controller:replication-controller
* system:controller:resourcequota-controller
* system:controller:route-controller
* system:controller:service-account-controller
* system:controller:service-controller
* system:controller:statefulset-controller
* system:controller:ttl-controller

