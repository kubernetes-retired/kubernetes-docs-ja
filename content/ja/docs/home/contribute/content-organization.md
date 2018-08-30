---
title: Content Organization
date: 2018-04-30
content_template: templates/concept
weight: 42
---

{{< toc >}}

{{% capture overview %}}

このサイトはHugoを試用しています。Hugoでは、[content organization](https://gohugo.io/content-management/organization/)がコアコンセプトとなっています。

{{% /capture %}}

{{% capture body %}}

{{% note %}}
**Hugo Tip:** Start Hugo with `hugo server --navigateToChanged` for content edit-sessions.
{{% /note %}}

## ページの一覧

### ページの順序

ドキュメントのサイドメニューで、ドキュメントのページブラウザなどがHugoでの規定の順序で一覧表示されます。規定の順序では、weight(1から)、日付(最新が最初)、タイトルの順でソートされます。

これを考慮すると、ページやセクションを上の方に移動したい場合、ページの前付けで、weightを設定します。

```yaml
title: My Page
weight: 10
```


{{% note %}}
ページのweightでは、1、2、3・・・という数列では無く、10、20、30・・・の様な別の間隔を使用した方が良いでしょう。この方法では、後にページを間に入れることができます。
{{% /note %}}


### ドキュメントのメインメニュー

「ドキュメント」メインメニューは`docs/`以下で、`_index.md`ファイルの前付けで`main_menu`フラグが設定されたセクションから構成されます。

```yaml
main_menu: true
```


リンク名はページの`linkTitle`から取得されるので、タイトルとは違うリンク名にしたい場合はコンテントファイル内で次のように変更してください。


```yaml
main_menu: true
title: Page Title
linkTitle: Title used in links
```


{{% note %}}
以上のことは言語毎に行われる必要があります。もしあなたのセクションがメニュー上で表示されていなかったら、おそらくHugoによってセクションとして認識されていないためです。セクションのフォルダ内に`_index.md`コンテントファイルを作成してください。
{{% /note %}}

### ドキュメントのサイドメニュー

ドキュメントのサイドバーメニューは`docs/`以下の _現在のセクションツリー_ から構成されます。

これは、すべてのセクションと、そのページを表示します。

セクションやページを表示したくない場合、前付けで`toc_hide`フラグを設定します。


```yaml
toc_hide: true
```

セクションにアクセスしたとき、セクションページ(例えば`_index.md`)に内容があればセクションページが表示され、内容が無ければセクション内の最初のページが表示されます。

### ドキュメントブラウザ

ドキュメントのホームページにあるページブラウザは、すべてのセクションと`docs section`直下のページから構成されます。

セクションやページを表示したくない場合、前付けで`toc_hide`フラグを設定します。

```yaml
toc_hide: true
```

### メインメニュー

右上のメニューとフッターに表示されるサイト内リンクはpage-lookupにより構成されます。これはページが実際にあるかどうかを確認します。そのため、もし`ケーススタディ`ページがサイト(あるいは選択された言語)に存在しない場合、リンクされません。

## ページバンドル

単体のコンテントページ(Markdownファイル)に加え、Hugoは[Page Bundles](https://gohugo.io/content-management/page-bundles/)をサポートしています。

例として、[Custom Hugo Shortcodes](/docs/home/contribute/includes/)があります。所謂`leaf bundle`です。`index.md`を含むディレクトリ以下のすべてのものはバンドルの一部となり、ページ相対リンクで画像等を処理したりできます。

```bash
en/docs/home/contribute/includes
├── example1.md
├── example2.md
├── index.md
└── podtemplate.json
```

広く使われているもう一つの例は`includes`バンドルです。これは前付けで`headless: true`と設定され、URLを持たないことを意味します。これは他のページ内でのみ使用されます。

```bash
en/includes
├── default-storage-class-prereqs.md
├── federated-task-tutorial-prereqs.md
├── federation-content-moved.md
├── index.md
├── partner-script.js
├── partner-style.css
├── task-tutorial-prereqs.md
├── user-guide-content-moved.md
└── user-guide-migration-notice.md
```

バンドルについて、いくつかの重要な注意点があります:

* 翻訳されたバンドルでは、発見できなかったファイルは継承されます。重複はしません。
* バンドル内のすべてのファイルはHugoで`Resources`と呼ばれ、パラメータやタイトルのように、たとえ(YAMLファイルなどのように)前付けをサポートしていなかったとしても、言語毎にメタデータを提供できます。詳細は[Page Resources Metadata](https://gohugo.io/content-management/page-resources/#page-resources-metadata)を参照してください。
* `Resource`から、`.RelPermalink`を得たときの値はページ相対です。


## スタイル

このサイト用の`SASS`ソースは`src/sass`以下に格納されており、`make sass`でビルドできます。 (Hugoは近いうちに`SASS`をサポートするようです: https://github.com/gohugoio/hugo/issues/4243).

{{% /capture %}}

{{% capture whatsnext %}}

* [カスタムHugoショートコード](/docs/home/contribute/includes)
* [スタイルガイド](/docs/home/contribute/style-guide)

{{% /capture %}}
