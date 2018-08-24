---
layout: docwithnav
title: Kubernetesドキュメントへのコントリビューション
---

<!-- BEGIN: Gotta keep this section JS/HTML because it swaps out content dynamically -->
<p>&nbsp;</p>
<script language="JavaScript">
var forwarding=window.location.hash.replace("#","");
$( document ).ready(function() {
    if(forwarding) {
        $("#generalInstructions").hide();
        $("#continueEdit").show();
        $("#continueEditButton").text("Edit " + "content/en/" + forwarding);
        $("#continueEditButton").attr("href", "https://github.com/kubernetes/website/edit/{{< param "docsbranch" >}}/" + "content/en/" + forwarding)
        $("#viewOnGithubButton").text("View " + "content/en/" + forwarding + " on GitHub");
        $("#viewOnGithubButton").attr("href", "https://git.k8s.io/website/" + "content/en/" + forwarding)
    } else {
        $("#generalInstructions").show();
        $("#continueEdit").hide();
    }
});
</script>
<div id="continueEdit">

<h2>編集を続ける</h2>

<p><b>ドキュメントの編集を続けるには、次の様にしてください:</b></p>

<ol>
<li>現在のページを編集するには、下部にあるボタンをクリックします。</li>
<li>あなたのGitHubアカウントに<i>fork</i>と呼ばれる、このサイトのコピーを作成するために、画面下部の<b>Commit Changes</b>ボタンをクリックします。</li>
<li>forkが作成されたら、forkに対して変更を加えることが出来ます。</li>
<li>変更について通知するために、インデックスページで<b>New Pull Request</b>ボタンをクリックします。</li>
</ol>

<p><a id="continueEditButton" class="button"></a></p>
<p><a id="viewOnGithubButton" class="button"></a></p>

</div>
<div id="generalInstructions">

<h2>クラウド上でサイトを編集する</h2>

<p>このサイトのリポジトリにアクセスするためには、下記のボタンをクリックしてください。画面の右上の<b>Fork</b>ボタンを押すと、<i>fork</i>と呼ばれる、リポジトリのコピーをあなたのGitHubアカウントに作ることが出来ます。あなたのforkに変更を加え、それらの変更を私たちに送る準備が出来たら、forkのインデックスページに移動して、<b>New Pull Request</b>ボタンを押してください。</p>

<p><a class="button" href="https://github.com/kubernetes/website/">このサイトのソースコードを表示する</a></p>

</div>
<!-- END: Dynamic section -->

<br/>

Kubernetesのドキュメントへコントリビュートすることについて詳細を知りたい場合、次のページを見てください:

* [Creating a Documentation Pull Request](/docs/home/contribute/create-pull-request/)
* [Writing a New Topic](/docs/home/contribute/write-new-topic/)
* [Staging Your Documentation Changes](/docs/home/contribute/stage-documentation-changes/)
* [Using Page Templates](/docs/home/contribute/page-templates/)
* [Documentation Style Guide](/docs/home/contribute/style-guide/)
* How to work with generated documentation
  * [Generating Reference Documentation for Kubernetes Federation API](/docs/home/contribute/generated-reference/federation-api/)
  * [Generating Reference Documentation for kubectl Commands](/docs/home/contribute/generated-reference/kubectl/)
  * [Generating Reference Documentation for the Kubernetes API](/docs/home/contribute/generated-reference/kubernetes-api/)
  * [Generating Reference Pages for Kubernetes Components and Tools](/docs/home/contribute/generated-reference/kubernetes-components/)
