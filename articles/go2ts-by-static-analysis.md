---
title: "Goの型から静的解析でTypeScriptを生成したい" # 記事のタイトル
emoji: "🎭" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["go", "typescript", "静的解析"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---

この記事は[Go 2 Advent Calendar 2020](https://qiita.com/advent-calendar/2020/go2) 6日目の記事です。

# 何を作ったか

先日、[Goの良さを好きなだけ語りたい](https://zenn.dev/articles/go-10-year-anniversary/edit) という記事を書き、その中でも軽く触れたのですがGoの型から静的解析でTypeScriptのinterfaceを生成する[go2ts](https://github.com/go-generalize/go2ts)というライブラリを作った、という話をしました。

そのライブラリをどう実装したのかについて書いていきます。

# go2tsとは
とても雑な説明をすると、Goの
```go
package main

import (
    "time"
)

type Status string

const (
    StatusOK Status = "OK"
    StatusFailure Status = "Failure"
)

type Param struct {
    Status    Status    `json:"status"`
    Version   int       `json:"version"`
    Action    string    `json:"action"`
    CreatedAt time.Time `json:"created_at"`
}
```

というコードからTypeScriptの
```typescript
export type Param = {
        action: string;
        created_at: string;
        status: "Failure" | "OK";
        version: number;
}
```
というinterfaceを生成してくれるライブラリです。jsonタグを見ます。

つまるところ、OpenAPIやgRPCを利用すればGoとTypeScriptで共通の型を利用できますが、それをもっと手軽にGoの型からTypeScriptのinterfaceを生成できたら便利そうだな、というのが開発した理由です。

類似のライブラリはあったのですが型解析のために一度コンパイルし、reflectを利用した型解析をしており一時ファイルを生成していました。そのためあまり使い勝手が良くなく、また実行に時間がかかるなどの問題があったため独自で作ることにしました。

# Goの静的解析
Goでは静的解析ツールを簡単に実装することができますが、linterを簡単に実装したい、という場合であれば [golang.org/x/tools/go/analysis](https://pkg.go.dev/golang.org/x/tools/go/analysis) というパッケージに沿って実装することで[golangci-lint](https://github.com/golangci/golangci-lint) に組み込めたり、解析自体はまとめて実行されるために高速で動作するなどと言った利点があります。

# How go2ts works
## packages
go2tsは、このパッケージも内部で利用している[golang.org/x/tools/go/packages](https://pkg.go.dev/golang.org/x/tools/go/packages)(以下packages)を利用しています。 packagesの特徴として、go.modまで見て型解析をしてくれます。つまり、解析しているモジュールの外部であっても型を読みにいき、Named TypeやType Aliasも辿って型を取得することができます。めっちゃ便利。外部依存としてgoコマンドは利用しているようなので注意が必要です。

簡単にサンプルプログラムを示すと、
```go
cfg := &packages.Config{Mode: packages.NeedFiles | packages.NeedSyntax}
pkgs, err := packages.Load(cfg, "go", "fmt")

if err != nil {
    panic(err)
}
if packages.PrintErrors(pkgs) > 0 {
    os.Exit(1)
}

packages.Visit(pkgs, nil, func(pkg *packages.Package) {
    fmt.Println(pkg.ID, pkg.GoFiles)
})
```
packages.LoadにConfigと対象のパッケージを指定するだけで解析されます。その後、(まぁfor-loopでも良いのですが、) packages.Visitを使うことで全てのpackageに対して解析を行うことができます。

## ASTの構築
構文と型の解析をpackagesが行なってくれたら、そのASTを辿ってTypeScript interfaceを生成するためのASTに変換していきます。

[github.com/go-generalize/go2ts/pkg/types](https://github.com/go-generalize/go2ts/blob/master/pkg/types) にそのための型を持っています。

全ての型はmapのkeyとして使えるかどうか、また型の文字列表現を返すメソッドを持ちます。

```go
// Type interface represents all TypeScript types
type Type interface {
	UsedAsMapKey() bool
	String() string
}
```

また、StringとNumberは冒頭の例でもあったようにUnion Typeとして使えるようにしています。その候補を追加するためのメソッドを持ちます。
```go
// Enumerable interface represents union types
type Enumerable interface {
	Type

	// AddCandidates adds a candidate for enum
	AddCandidates(v interface{})
}
```

また、structではそのstructのパッケージのパスと名称を設定するためのメソッドを持ちます。

```go
// NamedType interface represents named types
type NamedType interface {
	Type

	// SetName sets an alternative name
	SetName(name string)
}
```

typesパッケージでは
- Any
- Array
- Boolean
- Date
- Map
- Nullable
- Number
- Object
- String

の型を持っています。Mapはいわゆる辞書型で、Objectがstructに相当するものです。GoのinterfaceはAnyにキャストされます。

TypeScriptではnullとundefinedは区別されますが、nullはNullableで別の型をラップし、undefinedはObjectのフィールドの1つとしてoptionalフラグを立てる形で保持しています。

packagesのASTからgo2tsのtypesに変換する部分で踏んだバグについて一つ紹介したいと思います。

わかる人なら以下を見ればすぐにどんなバグかわかると思うのですが、Goでは以下のようにstructは自身のstructのポインタを持つことができます。

```go
type Foo struct {
    Child *Foo
}
```

解析器は再帰的にASTを辿って変換しており、上記のようなケースでは同一の型を繰り返し解析してしまうため永遠に解析が終わらずスタックオーバーフローが発生します。

go2tsのparserでは、解析した型をあるmapに代入していました。一度解析した型であればそのmapを参照して返すため何度も解析してしまうことはないのですが、子要素を全て解析してからmapに代入しているため再帰的に解析している間はmapに存在しておらず返すことができません。

そこで、structを解析している間は一旦mapにダミーの型を代入しておき、あとでその内容を書き換えるという若干無理やりな方法で解決しました。根本的で綺麗な解決ではないのであまり望ましいとはいえないのですが一旦はこの手法で対処しています。

## コード生成
コード生成部分は、基本的に先ほど生成したTypeScriptのASTを再帰的に辿って生成するだけで良いので特段面白いことはないのですが、コード自動生成において極めて重要なのは冪等性のあるコードを生成することです。冪等性がない場合、ツールを実行した際に関係ない部分に変更が発生してしまい、Pull Requestで余計な差分が発生しコンフリクト地獄になったりします。特にGoでは、mapをforで見ると自動的にランダムな順番で辿るようになっており、冪等性のないコードが生成されがちなので注意が必要です。

コード生成処理を書いてみて分かったのですが、今回のシンプルさもあり、再帰的に生成をするだけでインデントまで含めてかなり綺麗なコードになりました。

- Any → any
- Array → child_type[]
- Boolean → boolean
- Date → string
- Map → {[key: key_type]: value_type}
- Nullable → child_type | null
- Number → number
- Object → { field_name: child_type; ...}
- String → string

基本的には以上の変換を実施するのみです。また、Enumの要素を持っていた場合はstringやnumberの代わりに"a" | "b"のように生成します。

しかし、以上の規則では以下のような場合に誤ったコードが生成されます。
```go
type Foo struct {
    Array []*int
}
```

このコードでは以下のような型が生成されてしまいます。これはnumber型、もしくは"nullの配列型"を許容する型です。
```typescript
export type Foo = {
    Array: number | null[];
}
```

これを避けるため、enumもしくはnull許容の場合には()で囲むような処理を入れています。

# おわりに
Goの静的解析は非常に強力です。linterを書く際に使われることが多いですが、Goのコードから何かを生成することもできるよ！という例で紹介しました。

お読みいただきありがとうございました。
