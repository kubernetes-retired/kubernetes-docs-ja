---
title: DaemonSet（デーモンセット）
content_template: templates/concept
weight: 50
---

{{% capture overview %}}

_DaemonSet（デーモンセット）_ は、ポッドのコピー（複製）を全て（または一部の）ノードで実行します。
ノードがクラスタに追加されると、ポッドもそのノードに対して追加されます。
ノードがクラスタから削除されれば、ポッドも削除（ゴミ収集）されます。
DaemonSet を削除すると、DaemonSet によって作成されたポッドも削除されます。

DaemonSet の典型的な使い方は:

- クラスタ・ストレージ・デーモンを各ノードで実行。`glustered` や `ceph` など
- 全てのノードでログ収集デーモンを実行。 `fluentd` や `logstash` など。
- 全てのノードでノード監視用デーモンを実行。[Prometheus Node Exporter](https://github.com/prometheus/node_exporter)、 `collectd` 、Datadog エージェント、New Relic エージェント、Ganglia `gmond` など。

簡単な例としては、DaemonSet は全てのノードをカバーし、様々な種類のデーモンを動かすために使えます。
複雑なセットアップ例としては、１つのデーモンのために、複数の DaemonSet をセットアップします。
ただし、ハードウェアの種類によって、異なるフラグをつけたり、異なるメモリや CPU 要求を指定します。


{{% /capture %}}

{{< toc >}}

{{% capture body %}}

## DaemonSet Spec を書くには {#writing-a-daemonset-spec}

### DaemonSet の作成 {#create-a-daemonset}

DaemonSet は YAML ファイルで記述できます。
たとえば以下の `daemonset.yaml` ファイルで DaemonSet を記述する例では、fluent-elasticsearch Docker イメージを実行します。

{{< codenew file="controllers/daemonset.yaml" >}}

YAML ファイルをベースとした DaemonSet を作成するには:
```
kubectl create -f https://k8s.io/examples/controllers/daemonset.yaml
```

### 必要なフィールド {#required-fields}

他の Kubernetes 設定情報ファイルと同様に、DaemonSet では `apiVersion` 、 `kind` 、 `metadata`  フィールドが必要です。
設定情報ファイルの働きに関する一般的な情報は、[アプリケーションのデプロイ](/jp/docs/user-guide/deploying-applications/)、
[コンテナの設定](/jp/docs/tasks/)、[kubectl によるオブジェクト管理](/jp/docs/concepts/overview/object-management-kubectl/overview/)の各ドキュメントをご覧ください。

また、DaemonSet には [`.spec`](https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status) セクションも必要です。

### ポッド・テンプレート {#pod-tempalte}

`.spec.template` は `.spec` フィールドに必要な要素です。

`.spec.template` は  [ポッド・テンプレート](/jp/docs/concepts/workloads/pods/pod-overview/#pod-templates) です。
これは [ポッド](/jp/docs/concepts/workloads/pods/pod/) と完全に同じスキーマですが、ネスト化せず、 `apiVersion` と `kind` を持ちません。

ポッドに対して必要なフィールドに加えて、DaemonSet 内のポッド・テンプレートには適切なラベルの指定が必要です（詳細は [ポッド・セレクタ](#pod-selector) をご覧ください。）。

DaemonSet 内のポッド・セレクタは、 [`RestartPolicy`（再起動方針）](/jp/docs/user-guide/pod-states)が常に `Always` （常時）と同等であり、もし指定が無ければ、デフォルトで `Always`  になります。

### ポッド・セレクタ {#pod-selector}

`.spec.selector` フィールドはポッド/セレクタです。
これは [ジョブ](/jp/docs/concepts/jobs/run-to-completion-finite-workloads/) の `.spec.selector` と同じ挙動です。

Kubernetes 1.8 では、 マシンの`.spec.templaet` のラベルと一致するポッド・セレクタを指定する必要があります。
ポッド・セレクタは、デフォルトでは何も指定されていません。
セレクタのデフォルト化は `kubectl apply` と互換性がありません。
また、 DaemonSet が作成されても、 `.spec.selector` はマウントされません。
ポッド・セレクタの変更（mutating）によって、意図しないポッドの孤立をもたらし、見つけられなくなるのでユーザに対して混乱を与えるでしょう。

`.spec.selector` は２つのフィールドによって構成されるオブジェクトです。

* `matchLabels` - [ReplicationController](/jp/docs/concepts/workloads/controllers/replicationcontroller/) の `.spec.selector` と同じ挙動です。
* `matchExpressions` - より適切なセレクタによって構築できるようになります。関係するキーと値によって、キー、値のリスト、演算子（オペレータ）を指定します。

２つが指定されると、結果は AND として処理されます。

`.spec.selector` が指定されると、これは  `.spec.template.metadata.labels` と一致する必要があります。
設定が一致しなければ、API によって拒否されます。

また、通常はこのポッドに対するラベルは以下のものと一致すべきではありません。セレクタのみならず、他の DaemonSet を経由して作られたラベル、他の ReplicaSet のようなコントローラを通して作成されたものラベル。
そうしなければ、DaemonSet コントローラは、それぞれのポッドが各コントローラによって作成されたとみなされます。
Kubernetes は、この指定を止められません。
やむを得ず手動でポッドを作成したい場合は、テストのために異なったノードで実施してください。

### 同じノード上でのみポッドを実行 {#running-pods-on-only-some-nodes}

`.spec.template.spec.nodeSelector` を指定すると、DaemonSet コントローラは  [ノード・セレクタ](/jp/docs/concepts/configuration/assign-pod-node/) に一致するノード上でポッドを作成します。
同様に、.spec.template.spec.affinity` を指定すると、DaemonSet コントローラは [ノード親和性（node affinity）](/jp/docs/concepts/configuration/assign-pod-node/) に一致するノード上でポッドを作成します。
もし、どちらも指定しなければ、DaemonSet コントローラは全てのノード上でポッドを作成します。

## どのようにしてデーモン・ポッドがスケジュールされるのか {#how-daemon-pods-are-scheduled}

通常、ポッドを実行するマシンを選ぶのは Kubernetes スケジューラです。
しかしながら、DaemonSet コントローラによって作成されたポッドは、すでにマシンが選択されています（ポッド作成時に `.spec.nodeName` が指定されていても、スケジューラによって無視されます ）。
つまり:

  - DaemonSet コントローラによってノードの [`unschedulable`（スケジュールしない）](/jp/docs/admin/node/#manual-node-administration)は無視されます。
  - DaemonSet コントローラはスケジューラが開始されていなくても、クラスタのブートストラップの助けを得て、ポッドを作成できます。

(ToDo)

## デーモン・ポッドとの通信 {$communicating-with-daemon-pods}

DaemonSet がポッドと通信する可能性があるパターンは複数あります：

- **送信（push）**: DaemonSet 内のポッドは、データベースの状態など、他のサービスに対して更新情報を送るための設定が可能です。これらはクライアントを持ちません。
- ***ノード IP と既知ポート*: DaemonSet のポッドは `hostPort` を使えますので、ポッドにはノード IP を経由して到達できます。クライアントが何らかの手段でノード IP の一覧とポートが分かれば便利でしょう。
- **DNS**: 同じポッド・セレクタで [ヘッドレス・サービス（headless service）](/jp/docs/concepts/services-networking/service/#headless-services) を作成します。そして、DaemonSet は `endpoints` リソースや DNS の A レコードを代わりに使って発見できるようになります。
- **サービス**: 同じポッド・セレクタを使ってサービスを作成し、ランダムなノード上のデーモンにサービスを到達させます（到達先のノードを指定できません）。

## DaemonSet の更新 {#updating-a-daeomonset}

ノードのラベルを変更すると、DaemonSet は即座に一致するノードにポッドを追加し、一致しないノードからはポッドを削除します。

ポッドの調整は DaemonSet 作成時に行えます。
しかし、ポッドは全てのフィールドを更新できません。
また、DaemonSet コントローラは次回にノードが作成されるときはオリジナルの（元々あった）テンプレートを使います（たとえ同じ名前の場合であっても）。

DaemonSet は削除可能です。
もし `kubectl` で `--cascade=false` を指定すると、ポッドは対象ノードを離れます。
それから別のテンプレートを使って新しい DaemonSet を作成できます。
別のテンプレートを使う新しい DaemonSet は、全ての既存のポットが持っているラベルが一致するかどうか認識します。
ポッド・テンプレートが一致しないの変更または削除できません。
新しいポッドを強制的に作成する必用がある場合は、ノードまたはポッドを削除する必要があります。

Kubernetes バージョン 1.6 以降では、DaemonSet 上で [ローリング・アップデートを処理](/jp/docs/tasks/manage-daemon/update-daemon-set/) できます。

## DaemonSet の代替 {#alternatives-to-daemonset}

### Init スクリプト{#init-scripts}

ノード上で確実に実行可能なデーモン・プロセスを直接実行します（例： `init`、`upstard`、`systemd` の使用 ）。
この方法が完全に正しいです。
しかし、DaemonSet を経由するプロセスを使う方法に複数の利点があります。

- アプリケーションと同じ手法でデーモンにタンする監視やログ管理ができる
- デーモンとアプリケーションに対して同じ設定言語とツールを使える（例、ポッド・テンプレート、 `kubectl`）
- デーモンをコンテナ内で実行すると、リソースを制限し、デーモンとアプリケーション・コンテナ間の隔たり（isolation）を高めます。しかし、これによってデーモンはポッドではなく、完全にコンテナとして実行する必要があります（例：Docker を経由して直接起動）。

### ベア・ポッド（Bare Pods） {#bare-pods}

ポッドを作成するにあたり、特定のノード上での実行を直接指定できる場合があります。
しかしながら、DaemonSet は何らかの理由によって、ポッドを削除または停止して置き換える場合があります。たとえば、ノード障害やカーネル更新のような破壊的なノードのメンテナンスです。
このような理由のため、DaemonSet を使うよりは、個々のポッドを作成するほうが良い場合もあるでしょう。

### 静的なポッド {#static-pods}

kubelet が監視している特定のディレクトリ上にファイルを書いて、ポッドを作成できます。
これらは [静的ポッド（static pods）](/jp/docs/concepts/cluster-administration/static-pod/) と呼びます。
DaemonSet とは違い、静的なポッドは kubectl や他の Kubernetes API クライアントで操作できません。
静的ポッドは apiserver に依存しないため、クラスタの立ち上げ時の構成に役立つでしょう。
また、静的ポッドは将来的に廃止の可能性があります。

### デプロイメント {#deployment}

DaemonSet は  [デプロイメント](/jp/docs/concepts/workloads/controllers/deployment/) と似ています。
どちらもポッドを作成し、どちらのポッドも停止しないプロセスを持ちます（例：ウェブサーバ、ストレージ・サーバ）。

フロントエンドなど、ステートレス（状態を持たない）サービスのためにデプロイメントを使うと、スケールのアップ・ダウン時に複製数とローリング・アウト（展開）は、ポッド上で実行するよりも確実に制御するのが重要になります。
DaemonSet を使うにあたり重要なのは、ポッドのコピーが常に全てまたは特定のホスト上で動いており、他のポッドよりも速く実行する必要がある時です。

{{% /capture %}}
