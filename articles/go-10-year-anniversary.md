---
title: "Goの良さを好きなだけ語りたい" # 記事のタイトル
emoji: "🎉" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["go"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

# はじめに
サークル内部向けに書いていた記事ですがZennにも供養します。

Tsuzuと申します。[早稲田大学のデジタル創作サークルMISW](https://misw.jp)でSysAdとかWeb班長をしています。引退目前の身です。
Goが10周年らしいです。おめでとうございます。
私自身もGoを書き始めて4年半ほどが経過しました。
以前はC++をよく使っていたのですが最近ではWebフロントエンド以外の開発ではGoを利用することがほとんどです。
今回はKibela活性化の目的も含めてGoの良さを思う存分語ろうと思います。

HashiCorpというTerraform、Vaultなどを始めとする様々なGoで開発されたソフトウェアやSaaSを提供している会社があります。 HashiCorpが開発している一般に一番有名なツールはVagrantでしょうか。(VagrantはRubyですが)
 HashiCorpのCEOであるMitchell Hashimoto氏がGoの良さについてHacker Newsに書いているのでこちらも是非お読みください。余談ですが彼は日系二世のアメリカ人らしいです。[1]
https://news.ycombinator.com/item?id=24888283

彼の文章と被る内容は多いのですが私見も交えてより詳しく書いてみようと思います。

# Goの良さ
Goの良さは色々あるのですが大きく分けると ツールチェーン、言語仕様、ランタイムに分類されます。一つ一つみていこうと思います。

## ツールチェーン
まず他の良さについて語る上でも欠かせないのがツールチェーンの良さです。ツールチェーンとはビルドツールから始まり、linter、Language Serverなどがあります。

### ビルドツール
`go` コマンドはコンパイラもリンカーも内包しています。go buildでビルドを実行するとgo.modによる依存の解決からバイナリの生成まで実行します。多くのユースケースにおいてほとんどコンパイルオプションを意識する必要はありません。常に最適化はかかりますし大凡常に静的リンクです。

最近ではnetパッケージの一部がlibcに依存している部分があったりしますが、 `CGO_ENANBLED=0` の環境変数を付けてコンパイルすれば完全にlibc非依存となります。libc非依存にできることの最大の恩恵はクロスコンパイルの容易さです。GOOSやGOARCHの環境変数で様々なOS、アーキテクチャへクロスコンパイルが可能です。

goコマンドはビルドだけでなくフォーマッターも内包しており多機能です。これを https://golang.org/dl からダウンロードして展開するだけで良いので、環境構築が難しく困ることはほぼほぼないでしょう。

また、大体最新のバージョンを使っておけば困らないというのも嬉しい点です。[goenv](https://github.com/syndbg/goenv) などでバージョンを切り替えることもできますが、Goは後方互換性が強く各バージョンの安定性も高いためgoenvを使っていないという人も結構多いと思います。私は使っていません。

新しいバージョンが出ていてまだメインのGoをあげるのは怖いけど試してみたいという人は

```console
$ go get golang.org/dl/go1.15.3
$ go1.15.3 download
```
のようにするだけで特定のバージョンを試すことができます。 
参考: https://twitter.com/golang/status/1316490217591840769

### linter
Goは静的解析が盛んな言語です。golang.org/x/以下の準標準パッケージには静的解析を行うために必要なパッケージが充実しています。これらを利用して、様々なlinterが開発されています。

広く使われていて有名なlinterツールとして[golangci-lint](https://github.com/golangci/golangci-lint) があります。golangci-lintは様々なlinterをまとめて使えるようにしたもので、 `.golangci.yml` に書いた設定に沿ってlinterを実行してくれます。CI、特にGitHub Actionsとの[連携](https://golangci-lint.run/usage/install/#ci-installation)は非常に簡単ですのでオススメです。 ↓こんな感じ

![golangci-lint w/ GitHub Actions](https://github.com/golangci/golangci-lint-action/raw/853ade09ed4bc145ab0a4443299ab0536fea02fb/static/annotations.png)

出典: https://github.com/golangci/golangci-lint-action/blob/853ade09ed4bc145ab0a4443299ab0536fea02fb/static/annotations.png

ただ、golangci-lintは設定を書くのがかなり面倒なため社内で共通のテンプレート化されていることがほとんどでしょう。これを個人で用意するのは地味に面倒だったりします。もう少しシンプルでいいのなら`go vet` でもある程度のことができます。

自分で独自のlinterを書きたい場合は golangci-lintの[How to write a custom linter](https://github.com/golangci/golangci-lint/blob/master/docs/src/docs/contributing/new-linters.mdx)を参考にすれば書けます。golangci-lintのcustom linterは[golang.org/x/tools/go/analysis.Analyzer](https://pkg.go.dev/golang.org/x/tools/go/analysis#Analyzer) をベースとして作成します。これをベースとすることでCLIにしたり他のツールとの連携が楽になります。


最近私はGoのstructからTypeScriptを生成するライブラリ[go2ts](https://github.com/go-generalize/go2ts)を書いたのですが、これは解析に[golang.org/x/tools/go/packages](https://pkg.go.dev/golang.org/x/tools/go/packages)を利用しています。go.modまでみて型解決も行ってくれるので複雑な解析には便利です。go2tsについてはまた今度記事を書きたいですね。

もっと静的解析について知りたい方は[tenntennさん](https://tenntenn.dev/ja/analysis/)がそのあたりの情報を沢山発信されているので参考になります。mercari Gophers Office Hoursでも[静的解析について取り扱っている回](https://youtu.be/ER3HtesHfWM)がありました。Goにおけるlinterはコーディング規約を守らせるという側面ももちろんありますが、バグを防ぐためのlinterとしての側面が大きいのが特徴です。代表的なところでいうと、 `io.Closer`の閉じ忘れ等。Goは型システムがそこまで強くない代わりに静的解析によって安全性を担保する仕組みが充実しています。

また、コード生成の仕組みも充実しており、詳細は省略しますがモックやOpenAPI、GraphQLサーバなども自動生成します。

### Language Server
[gopls](https://pkg.go.dev/golang.org/x/tools/gopls)(発音はGo Pleaseっぽい?)という公式で開発されているLanguage Serverがあり、比較的新しいですが安定してきています。各種エディタにLSP対応の拡張を入れれば快適に動作します。シンプルな言語仕様故、補完が効かない、みたいなことはほとんどありません。

### pkg.go.dev
以前はWebでドキュメントを読むのは[godoc.org](https://godoc.org)でしたが今ではGo Modulesに対応した[pkg.go.dev](https://pkg.go.dev)に移行しつつあります。ソースコード上に書いたコメントなどをまとめて見やすくしてくれるやつです。

## 言語仕様
### 複雑でない言語仕様
Goの言語仕様のシンプルさが故の問題点としてよくエラーハンドリングの煩雑さが槍玉に上がります。Goにおけるエラーハンドリングの仕組みを軽く説明しておきます。知っている方は読み飛ばしてください。

Goではいわゆる例外と似た仕組みとしてpanic/recoverというものがあり、以下のように利用できます。deferは関数の終了時に自動で呼び出されるハンドラを登録するものです。予めdeferで登録しておかなければエラーを受け取ることができません。

```go
func fn() {
	panic("something went wrong...")
}

func main() {
	defer func() {
		if e := recover(); e != nil {
			fmt.Println("recovered:", e)
		}
	}()
	
	fn()
}
```

```出力
recovered: something went wrong...
```
参考: https://play.golang.org/p/w4Pnz4ZcMi0

配列外参照などが起こった場合panicが発生しますが基本的にpanic/recoverを利用したエラーハンドリングは推奨されません。関数は複数の戻り値を返すことができ、その最後としてerrorを渡すことが基本的な手法です。

```go
func main() {
	resp, err := http.Get("https://example.com/")

	if err != nil {
		fmt.Fprintf(os.Stderr, "something went wrong: %+v", err)

		os.Exit(1)
	}

	fmt.Println("The response code is", resp.Status)
}
```

```出力
something went wrong: Get "https://example.com/": dial tcp: lookup example.com on 169.254.169.254:53: dial udp 169.254.169.254:53: connect: no route to host
Program exited: status 1.
```

何か関数を呼び出すたびに以下のようなコードを書くことになります。
```go
if err != nil {
    return nil, err
}
```
実際にはこのコードも微妙で、 `fmt.Errorf` や `errors.Wrap` でerrorをラップしスタックトレース情報を付加させるのが通例です。
```go
if err != nil {
    return nil, fmt.Errorf("internal error: %+w", err)
}
```
この書き方が面倒で無駄だ、というのがよく批判されている理由です。実際初期化処理など、あまりに大量に書く場所では面倒と感じることもあります。このように書き手に面倒を強制する仕組みはエラーをユーザに明示的に処理させるために、つまりエラーを無視させないために導入されています。ただ、これが最適な方法であるというわけではなく、今後他のハンドリング手法が導入される可能性も、もちろんあります。

将来的に導入されるかもしれない大きな機能に関する議論は[GitHubのissue](https://github.com/golang/go/issues?q=is%3Aopen+is%3Aissue+label%3AGo2)で[Go 2](https://blog.golang.org/go2-next-steps)として議論されています。Goは後方互換性を極めて重視しているため慎重に議論されており、なかなか先進的な機能が導入されないのは事実です。 とはいえGo 2はGo 2.X.Yで導入される、というわけでもなく近い将来導入される可能性もあります。

実際、エラーハンドリングと並べてよく槍玉にあがる[ジェネリクス](https://blog.golang.org/generics-next-step)は順調にいけば2021年8月リリース予定のGo 1.17で導入されるかも、と言われています。ジェネリクスは普段の開発ではなくても事足りますが、たまにデータ構造などを作る場合には欲しいなぁと実感します。

Goの言語仕様として全体的に重視されているのはコードの読みやすさです。コードの読みやすさとは誰が読んでも何が書いてあるかわかる、ということをさします。誰が書いても良いコードが書ける、は流石に厳しいですが、初学者であっても上級者が書いたコードを読んで何をしているか理解するのは容易でしょう。ライブラリの内部実装も気軽に読めます。手元にcloneして解析するのが面倒であれば[Sourcegraph](https://about.sourcegraph.com/) を使えばWeb上でもJump to definitionできます。(GitHubの機能にも同様のものがありますが割と頻繁に動作しないのでSourcegraphの方が確実です。)

### 標準ライブラリとinterface
Goのinterfaceはダックタイピングを採用しており、interface側で宣言されたメソッドが定義されていればstruct側で明示的にそのinterfaceを満たしていることを明記する必要はありません。以下の例では `String() string` は `fmt.Stringer` で必要なメソッドです。明示的に struct `Hello` が `fmt.Stringer` を満たしていることは示していませんが、 `print` 関数に `fmt.Stringer` として渡すことができています。

```go
type Hello struct{}

func (h *Hello) String() string { // For fmt.Stringer
	return "hello"
}

func print(s fmt.Stringer) {
	fmt.Println(s.String())
}

func main() {
	print(&Hello{})
}
```

```出力
hello
```

Goでは標準ライブラリで様々なinterraceが定義されていますが、その中でも特に利用頻度が高いのは `io.Reader` 、 `io.Writer` 、 `io.Closer` の3つでしょう。

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
type Writer interface {
	Write(p []byte) (n int, err error)
}
type Closer interface {
	Close() error
}
```

極めてシンプルなこれらのinterfaceがなんの意味を持つのか、と思うかもしれません。しかし、これらが標準ライブラリで定義されていたからこそ、標準ライブラリからサードパーティライブラリまでがIOに関してはこれらの interface を満たして実装されています。 HTTPレスポンスのボディをJSONデコーダに流すこともjpeg/pngのデコーダに流すのも同一のインターフェースを介して実装できます。間にミドルウェアを挟むこともできます。ライブラリの初期化などでデータがメモリ上にあるのにファイルパスしか引数で取れない、などでは使い勝手が悪いのでこういった共通のインターフェースが定義されているのです。

強いていうのであれば `io.Reader` 、 `io.Writer` が引数でcontext.Contextをとってくれるととても便利だった気がするのですがこれはcontextが入るよりも先に `io.Reader` があったため難しいかもしれません。

これ以外にも `net.Conn` を始めとする様々なinterfaceが定義されています。もちろんガワ(interface)だけではなく、標準ライブラリと準標準の `golang.org/x/` を合わせるとかなりの数のライブラリがあります。

### UTF-8
細かい話ですがデフォルトの文字コードがUTF-8になっているのは文字コードで苦しめられてきた身としてはとても感動しました。

### モジュール
これは言語仕様というよりツールチェーンよりの話ですが、中央集権のパッケージマネージャでなくGitをベースとしたモジュールの依存解決の仕組みを積んでいます。どちらが良いかはなんとも言えませんが、既存の仕組みを利用して気軽にライブラリを公開できるので個人的にはアリだと思っています。ちなみに最近ではgo getの高速化のため[Go Modules Proxy](https://proxy.golang.org/) が利用されていますが、これは自分でも運用でき、また切っても問題なく動作するものです。

## ランタイムの良さ
Goにはランタイムがありますが、このランタイムがかなり優秀です。
### GC
GoのGCについては詳細な仕様をそこまで把握てきていないので詳しい説明はできないのですが、バージョンが上がるたびに繰り返し改良が加えられており、今ではStop The Worldの時間も極めて短くなっています。極限のスピードが求められる環境での利用は厳しいですが、Goの速度で許容できる、という場面も多いのではないかと思います。

### Goroutine
GoroutineはGoランタイムが管理してくれる軽量スレッドです。OSスレッドよりも1スレッドあたりのオーバーヘッドが小さいながら、ランタイムが自動的に実行するgoroutineを切り替えてくれて、複数コアを効率的に利用できます。

使い方はとても簡単で`go` に続いて関数呼び出しをするだけです。先ほどの`io.Reader`や`io.Writer` が同期的なインターフェースで十分であるのも、IOで待ちが発生するとランタイムが他のgoroutineに切り替えるといったスケジューリングを行ってくれるためです。

```go
func main() {
    go func() {
        fmt.Println("Hello, world from goroutine")
    }()
    time.Sleep(1 * time.Second)
}
```


# おわりに
Goの良さを思う存分語ることができて非常に満足しました。間違いなどあればご指摘いただけると幸いです。

![Gophers](https://storage.googleapis.com/zenn-user-upload/vwt9h4rukq4ocvstcf2zm7c26fpb)

[1]: https://www.publickey1.jp/blog/17/hashicorp_interview02.html
