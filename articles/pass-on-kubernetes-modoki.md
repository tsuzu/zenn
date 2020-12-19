---
title: "Kubernetes上で動くPaaSを作りたい" # 記事のタイトル
emoji: "👨‍👦" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["kubernetes", "paas"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---

:::message
この記事は[Kubernetesアドベントカレンダー2020](https://qiita.com/advent-calendar/2020/kubernetes) 20日目の記事です。
:::
# はじめに
色々あってPaaSを作ってみたいと感じ今年の夏ごろからKubernetes上で動くPaaSの開発を始めました。PaaSといってもknativeのようなものではなく、いわゆるPaaSといって想像するHerokuやCloudfoundryなどに近い抽象度のものです。CloudfoundryはKubernetesで動かすプロジェクトなどもあったりしますが根っこからKubernetes自体の思想にそったPaaSを作ってみたいという思いから開発を始めました。

まだ開発が進まずPoCレベルですが[modoki-paas/modoki-operator](https://github.com/modoki-paas/modoki-operator) で実装しています。9月ごろには[技育展](https://talent.supporterz.jp/geekten/2020/)で発表してました。

今回はその基本コンセプトと実装に当たってのコツみたいなところについて書かせていただきます。

# 基本設計
今回のPaaSの根幹はKubernetesのCRD及びコントローラとして実装しています。

今のところCRDは以下の3つです。
- Application
- RemoteSync
- AppPipeline

## Application
Applicationはアプリケーションとしてデプロイされる1単位です。Applicationリソースはアプリケーションの運用に必要なDeploymentやService、Ingressなどの管理します。Applicationリソースのspecとしてイメージやコマンドなどを受け取りそれをもとに展開します。

### Deployment、Service、Ingressなどの子リソース
PaaSにて様々なリソースを作成できるようにしたいという思いがありましたが、どういったリソースを作成するかをGoのコード内に埋め込んでしまった場合作成するリソースを環境によって変える場合、コードやイメージをその度にコンパイルし直す必要が出てきてしまいます。これを避けるために[cdk8s](https://github.com/awslabs/cdk8s)というツールを利用しています。

cdk8sはAWS CDKのKubernetes版に相当するものです。AWS CDKではソースコードでリソースを定義するとCloud FormationのYAMLが作成されますが、cdk8sではKubernetesのYAMLが生成されます。例えば以下のような感じです。(動作は未確認です。)

```typescript
class MyChart extends Chart {
    constructor(scope: Construct, name: string) {
        super(scope, name);

        const labels = {app: 'hello-k8s'};

        new Deployment(this, 'deployment', {
            spec: {
                replicas: 2,
                selector: {
                    matchLabels: labels,
                },
                template: {
                    metadata: { labels },
                    spec: {
                        containers: [
                            {
                                name: 'hello-kubernetes',
                                image: 'paulbouwer/hello-kubernetes:1.7',
                                ports: [ { containerPort: 8080 } ],
                            },
                        ],
                    },
                },
            },
        });
    }
}
```
cdk8sのChartのソースファイルをConfigMap経由でコントローラの起動時に読み込むようになっています。

(cdk8sはKubernetesのYAMLそのまま、といった感じですがcdk8s+というより抽象度の高いものも開発されています。これには未対応です。)

## RemoteSync
 RemoteSyncはその名の通り外部にあるリソースと同期を行います。基本的にはGitHubの特定のリポジトリの特定のブランチやPRに結びつきます。イメージをビルドするため、[kpack](https://github.com/pivotal/kpack)のImageリソースを作成し、ビルドが終わるたびに関連付けられたApplicationリソースのイメージを更新します。また、外部からのWebhookを受け付け、Webhookがきたタイミングでも更新がないかどうか確認するようにしています。

### kpack
Applicationリソースでは、Kubernetesの様々なYAMLをアプリケーション開発者側が書く、といった手間を削減することはできたのですが、Kubernetesにアプリケーションをデプロイするにはコンテナイメージが必要です。コンテナイメージは普通Dockerfileを利用してビルドしますが、今回はHerokuなどに準じてそれも無くしたいという思いがありました。

DockerfileなしでのイメージのビルドにはCloud Native Buildpacks(以下CNB)を利用しています。CNBは元々はHeroku BuildpacksやPivotal Buildpacksだったものが、CNCF Sandboxの元で標準化されたものです。もちろんDockerfileを書かなくて良いだけでなく、全てのイメージにベストプラクティスを適用することができるのでイメージサイズ削減などにもつながります。

[Paketo Buildpacks](https://paketo.io/)や[Google Cloud Buildpacks](https://github.com/GoogleCloudPlatform/buildpacks) などいくつかbuildpacksが公開されているのでこういったものを利用すれば自分で作成する必要はありません。

手元でCNBを使いたい場合は[pack](https://github.com/buildpacks/pack) CLIというツールが最もお手軽で多機能ですが、Kubernetes上でCNBを利用したイメージのビルドを行いたい場合に登場するのが[kpack](https://github.com/pivotal/kpack)です。kpackはKubernetesのCRDとしてビルドしたいイメージの状態を定義すると、自動的にイメージのビルドを走らせ、レジストリにpushまで行ってくれます。kpack側にリポジトリが更新された場合は自動的に再ビルドする機能があるのですが、これは1分おきで固定されており、Webhookでビルドするといったことができないため、Applicationリソースから明示的に特定のrevisionをビルドするよう指定しています。

## AppPipeline
AppPipelineはHerokuのAppPipelineからそのまま名前を拝借しています。これはGitHub上で作成されたPull Requestを自動的にデプロイするものです。Pull Requestがデプロイされれば作成者もレビュアーが動作確認がしやすい、というのが主な目的です。

これはRemoteSyncリソースとApplicationリソース双方を自動的に管理し、Pull Request関連のWebhookに応じてコントローラが走り自動的にそれらのリソースを作成、削除します。それぞれのPRに追加のcommitがあった場合などはAppPipelineは何もせず、RemoteSync側のコントローラがWebhookを受けます。AppPipelineはPRがopenかcloseかの状態を監視します。

以上のリソースをまとめると↓こんな感じになります。

![](https://storage.googleapis.com/zenn-user-upload/4gcpdvxmkg85whhdkhcrr5ydoxk9)

## ghapp-controller
PaaS自体のコンポーネントではないですが、[ghapp-controller](https://github.com/modoki-paas/ghapp-controller)はGitHub Appから特定のInstallation(UserもしくはOrganization)向けのトークンを発行するコントローラです。Installationトークンは1時間で失効しますが、自動的に更新するようになっています。PaaSがGitHubからpullをしてくるための認証としてGitHub Appを使う必要があったのですが、当初これはPaaSのコントローラで管理していました。しかし、想像以上に複雑になってしまったので外部のコントローラとして切り出しました。これによりPaaS側では認証情報はSecretリソースから読み出すだけと、かなりシンプルな構成になりました。ghcr.ioなども登場してきたので別のコントローラとして切り出したのは良かったかなと思っています。

# 実装におけるTips
ここからは実際にPaaSを実装してみて得られた知見などをいくつか紹介します。

## ControllerでWebhookを受け取りたい
Kubernetes上で動かすコントローラで今回のように外部からWebhookなどのイベントを受けてReconciliationを実行したい場合、どうやってイベントを受け取るべきか悩みました。
まず考えられたのはControllerManagerのWatchesにsource.Channelを渡す方法です。

operator-sdkの手法に沿って実装していた場合はReconcilerのSetupWithManagerで以下のように渡します。これでchannelに渡されたオブジェクトはReconciliation QueueにEnqueueされます。
```go
func (r *HogeReconciler) SetupWithManager(mgr ctrl.Manager, ch <-chan event.GenericEvent) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&modokiv1alpha1.Application{}, builder.WithPredicates()).
		Watches(&source.Channel{Source: ch}, &handler.EnqueueRequestForObject{}).
		Complete(r)
}
```

しかし、この手法には問題があります。このような手法で実装した場合、おそらくわざわざPubSubを導入したりしないでしょうから、コントローラでWebhookを直接受けることになります。コントローラを高可用性のために複数Pod起動していると、WebhookがLeaderでないPodに飛んでいってしまう可能性があります。これでは困るため別の手法を検討します。

同じようなことをやっているツールとしてArgoCDがあります。ArgoCDの実装を見てみましょう。ソースコードを読んでいくと以下の場所でWebhookで受けたイベントに応じてApplicationのRefreshAppを実行していることがわかります。

https://github.com/argoproj/argo-cd/blob/97003caebcaafe1683e71934eb483a88026a4c33/util/webhook/webhook.go#L231

RefreshAppは以下に実装がありました。

https://github.com/argoproj/argo-cd/blob/e3e392c058e11b1d0b7cd0d494f73e40ca8ec54f/util/argo/argo.go#L82

行っていることは非常に単純で、対象のリソースに対してPatchを実行しているだけです。KubernetesではPatchを実行すると実際のリソースと差分があるかないかに関わらずReconciliationが実行されます。この仕様の影響で私はServiceAccountに毎度Patchを当て大量のSecretが生成されるということがありました。皆さん気を付けましょう・・・。

## WebからKubernetes APIを叩く
このPaaS実装において、operatorだけでは他のPaaSにあるような使い勝手が不足していました。その原因としてはWebから見られるWebから確認できるダッシュボードの無さがあります。しかし、PaaSの根幹をKubernetes-nativeにした手前、Webページに複雑な権限管理機構やAPIを実装するというのは気が引けました。そこで、Web上から直接KubernetesのAPIを利用してWeb上に表示することを考えました。

KubernetesのJavaSciprt/TypeScriptのクライアントライブラリは[kubernetes-client/javascript](https://github.com/kubernetes-client/javascript)にありますが、問題はこのクライアントはサーバサイドしか対応していないことです。ただ、このJavaScriptライブラリは一部のメソッドを除き、[kubernetes/kubernetes](https://github.com/kubernetes/kubernetes)の[openapi-spec](https://github.com/kubernetes/kubernetes/tree/master/api/openapi-spec)から自動生成されています。他の言語に関しても自動生成されているクライアントは結構あります。そのためのスクリプトなどは[kubernetes-client/gen](https://github.com/kubernetes-client/gen)にまとまっています。

kubernetes-client/genの設定ファイルをもとに[modoki-paas/kubernetes-fetch-client](https://github.com/modoki-paas/kubernetes-fetch-client)にkubernetes-clientを自動生成するようにしました。GitHub Actionsの設定を見るとわかりますが素直に生成するだけではダメで無理やり生成ファイルにパッチを当てて使っています。

この時、生成に利用していた[kubernetes/kubernetes](https://github.com/kubernetes/kubernetes)ではOpenAPIのspecには当然本来CRDが含まれていません。しかし、実際にはCRDでもOpenAPIのspecを使ってクライアントを書きたいという気持ちになります。そこで、OpenAPIをCRDを導入してあるKubernetesクラスタから引き抜くことにしました。modoki-operatorのCIにて実際にkindにクラスタを構築し、/openapiエンドポイントからOpenAPIのspec取得し、[modoki-paas/kubernetes-openapi-generated](https://github.com/modoki-paas/kubernetes-openapi-generated)にGitHub Actions経由で作成するようにしてあります。

また、ブラウザから直接叩く場合、KubernetesのAPI Serverは基本的に独自のCAで署名しているのでブラウザから直接繋いでもアクセスできなかったり、署名の問題がなくてもクロスオリジンになってしまいます。そのため、苦肉の策としてプロキシを置いています。Kubernetesの認証としてOpenID Connectを利用している場合はWebだけで認証フローが完結できるようにしてあり、発行したトークンは暗号化してCookieに保存しています。

# 抱えている課題
## GitHub周りの認証機構
GitHubのtokenをghapp-controllerによってSecretリソースに保存するようにしてあります。このコントローラではGitHubAppとInstallationという2つのリソースを作成し、前者がどのGitHub Appであるか、後者がどのInstallation(UserやOrganization)であるかを定義しますが、後者を作成できるユーザは前者のGitHub Appで利用可能な全てのリポジトリへのアクセス権限を得てしまいます。対応策としてはInstallationリソースの作成を制限するしかないのですがこれをOAuth 2.0の認証フロー後に自動的に作成する機構も必要となり管理が大変になります。

また、一度Installationが作成されてしまえばそのnamespaceでSecretを参照できる人であれば誰でもそのトークンを取得できてしまうのも問題になると思っていますが、現状では解決策は見いだせていません。

# まとめ
Kubernetes上で開発するPaaSの設計の仮設計を紹介させていただきました。認証周りなどでもっと良い設計がないか検討中ですのでもしあればご提案いただけると幸いです。
