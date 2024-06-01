---
title: "chevrotainで順序を保持するINIパーサ・JSONパーサを作ってJSRで公開"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['パーサ', 'chevrotain', 'jsr', 'ini', 'json']
published: true
---

# TL;DR

chevrotainを使ってINIファイルとJSONファイルのパーサを作りました。JSRで公開しています。

https://jsr.io/@gunseikpaseri/perfect-ini-parser

https://jsr.io/@gunseikpaseri/perfect-json-parser


# 動機

既に、[@std/ini](https://jsr.io/@std/ini) [@std/json](https://jsr.io/@std/json)を始め、信じられない数のパーサが公開されています。
しかし、いずれも設定さえできればよいと平易なものばかり。ご丁寧に保存時に順序を揃えたり整形してします。

現在別件で様々なiniファイルやjsonファイルを統合管理できるようなツールを試作中なのですが、このとき差分が少ない方が嬉しく、現状それをやってくれるパーサが見当たらない（**多すぎて探す気になれなかった**）のです。

こんだけあるんだろうから、まああるんでしょう。例えば[json-file-plus](https://www.npmjs.com/package/json-file-plus/)はそれを謳っています。しかし、このパッケージはファイル操作も込みで少し扱いにくいです。

他にも意図しない変更をしないとか、色々条件を重ねていった結果、自分で作ってしまった方が早いかもな、という気持ちになってきました。

# chevrotain

さて、パーサを書くにも色々あります。
ゼロから実装するか、既存のパーサジェネレータを使うか、その手のサブモジュールを使うか。

- peg.js(メンテナンスされず) => [peggy](https://peggyjs.org/) + [ts-pegjs](https://github.com/metadevpro/ts-pegjs)
- [jison](https://gerhobbelt.github.io/jison/docs/)
- [loquat](https://github.com/susisu/loquat)
- [chevrotain](https://github.com/Chevrotain/chevrotain)

私は前にPEGを使ったことがあるので、PEG.jsやTypeScript対応のts-pegjsを使うつもりでした。それで実際途中まで書いてみました。ただ、[chevrotain](https://chevrotain.io/)というものがあるらしく、これが速いらしくまたGithubStarも多い(2.4K)ので、試してみることにしました。

# chevrotainについて

pegjsはパーサジェネレータであり、pegjsファイルを作成し、pegjsでファイルスクリプトをコンバートして出力されたファイルを使います。

一方でchevrotainはパーササポーターとでも言うべきか、chevrotainの文法でパーサを書くと、それがそのまま動きます。
正直ファイルサイズをまったく気にしていなかったのですがchevrotainのunpacked sizeは1.35MBあります。今後配布すること等を考えると、必要最低限のコードが生成されるコードジェネレータの方が良かったのかもしれませんが……。

表現できる文法はLL(K)文法であり、PEGのような優先順位の考慮はしてくれず、文法が一意に定まらない可能性があります。

## chevrotainの使い方

大きく字句解析と構文解析の2つに分かれます。字句解析で元のファイルを字句に抽象化し、構文解析でその組み合わせを解析します。

実際に[PlayGround](https://chevrotain.io/playground/)の文法を見ればよいです。しかし後述しますが、ここで行われている書き方だとTypeScript関係で少し不便があります。

### 字句解析
もとの文をトークンに分割します。例えば改行だとこんな感じです。
```ts
const LF = createToken({ name: "LF", pattern: "\n", label: "\\n" });
```
正規表現も使えます。
```ts
const WhiteSpace = createToken({
  name: "WhiteSpace",
  pattern: /\s+/,
});
```

作成したTokenを基に`Lexer`クラスを作成します。
```ts
const jsonTokens = [
  WhiteSpace,
  NumberLiteral,
  StringLiteral,
  RCurly,
  LCurly,
  LSquare,
  RSquare,
  Comma,
  Colon,
  True,
  False,
  Null,
];

export const JsonLexer = new Lexer(jsonTokens);
```

## 構文解析

構文解析は`CstParser`や`EmbeddedActionsParser`クラスを継承して作成したクラスの中で行います。
ただ構文木を作るだけなら`CstParser`、いい感じにJavaScriptを埋め込み独自の構文木を作るなどしたい場合は`EmbeddedActionsParser`を使います。
`EmbeddedActionsParser`を使うと、例えば数式を与えて計算させる、みたいなところまでをこのクラスの中で実現できます。

構文ルールをルールメソッドで定義します。例えば、`False`トークンと`Null`トークンをいずれかにマッチする`value`というルールを作成するとこんな感じです。
```ts
class Parser extends EmbeddedActionsParser {
  constructor() {
    super(jsonTokens, { recoveryEnabled: true });
    this.performSelfAnalysis();
  }
  public value = this.RULE("value", () => {
    this.OR([
      { ALT: () => this.CONSUME(False) },
      { ALT: () => this.CONSUME(Null) },
    ]);
  });
}
```

`CONSUME`はトークンを消費するメソッドです。`OR`はいずれかにマッチするメソッドです。

他にも、0回か1回を表す`OPTION`、0回以上を表す`MANY`、他のルールを使う`SUBRULE`、arrayのリストのようなセパレータを挟んだ複数個にマッチする`MANY_SEP`などがあります。
（基本的には`OPTION`・`MANY`・`SUBRULE`ですべて表現できると思いますが、いろいろ使うことで簡略化できそうです）

:::message
同階層で同じトークンを利用した処理を実行する場合、内部処理で区別がつかなくなってしまうようです。
そのため、二回目以降は`SUBRULE1`、`SUBRULE2`といった形で指定する必要があります。
（基本的に問題があればエラーを出してくれます。）
:::

:::message
前から解析する都合上、ORのように分岐発生時、同じものにマッチするルールが先頭にある場合にも通らなくなってしまうことが多いので、前方に関しては括り出しておくとよいです。
:::

ちなみに元の文では全てコンストラクタ内で定義されていますが、`SUBRULE`等を使う際、型解決できないため、`public`でひとつずつ指定する様式になっています。

`RULE`メソッドの戻り値で何らかのオブジェクトを指定すると、`this.SUBRULE()`や`OR`がそのオブジェクトを返してくれます。また、`this.CONSUME().image`でマッチした文字列を取得できます。
これを利用して、独自の解析結果を返してあげましょう。

## 使い方

パーサのルールを呼び出せばよいです

```ts
const parser = new JsonParser();

function parse(text: string) {
  const lexingResult = JsonLexer.tokenize(text);
  parser.input = lexingResult.tokens;
  return parser.json();
}
```

パーサは`parser.input`を更新するたびにリセットされるようです。ここに字句解析結果を代入した上で、構文解析で`public`で公開したルールを呼び出すと、構文解析結果を得ることができます。


## 構文図の表示

[こんな感じ](https://chevrotain.io/diagrams_samples/json.html)の構文図を表示することができます。（Railroad図と言うらしいです）

コード側でやっていることは、chevrotainが用意したHTMLファイルに文法のjsonを埋め込んでいるだけです。

内部的には[railroad-diagrams](https://github.com/tabatkins/railroad-diagrams)というライブラリを使っているようです。

```ts
import { createSyntaxDiagramsCode } from "chevrotain";

import { IniParser } from "./src/ini_parser.ts";

const parser = new IniParser();

const grammar = parser.getSerializedGastProductions();
const html = createSyntaxDiagramsCode(grammar);

Deno.writeTextFile("./ini_syntax_diagram.html", html);
```

GitHub上で見ずらいので、mermaidとかに出力してくれると嬉しいんですが、進捗は芳しくないですね……

https://github.com/mermaid-js/mermaid/pull/4608

# JSRで公開

解析結果を何やかんやして、動くツールになったので、JSRで公開しました。

JSRは、npmやdeno.landの代替であり、esmの便利機能が色々使え、npm・bun・denoで動作します。

例えばコード内に書き込んだJSDocがブラウザから確認出来たり、ドキュメント状態などをスコア化して評価指標として出力してくれたり、Deno・Node.js・Bun・ブラウザそれぞれのサポート状況を明示させることができます。

一方、普及率の問題もあるのか、WeeklyDownloadが表示されません。
また、Homepageセクション・Licenseが無いです。（これはなんとか増やしてほしいですね）

GitHub Action経由で公開させることで、安全性を保障していることをアピールできます。

## 公開手順

JSRはスコープが必須化されているため、アカウント登録後に自分用のスコープを作成します。自分は`@gunseikpaseri`としました。

公開に当たって、`deno.jsonc`ファイルに情報を書き込みましょう。`name`には公開するスコープを含めた名前を指定します。`exports`には公開するファイルを指定します。`publish`には公開するファイルを指定します。`include`ではなく`exclude`を使うこともできます。

```json
{
  "name": "@gunseikpaseri/perfect-json-parser",
  "version": "0.0.0",
  "exports": "./mod.ts",
  "publish": {
    "include": [
      "deno.jsonc",
      "mod.ts",
      "README.md",
      "src/",
      "!src/**/*.test.ts"
    ]
  },
}
```

GitHubアクションを作成しましょう。tag設定時に、それに従ったバージョンが公開されるようにするには、[@david/publish-on-tag](https://github.com/dsherret/jsr-publish-on-tag)を利用するとよいです。作成したdsherretさんはDenoの人なので大丈夫でしょう。

```yml
name: jsr

env:
  DENO_VERSION: 1.x

on:
  push:
    tags:
      - "v*"

permissions:
  contents: read
  id-token: write

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: denoland/setup-deno@v1
        with:
          deno-version: ${{ env.DENO_VERSION }}
      - name: Publish
        run: |
          deno run -A jsr:@david/publish-on-tag@0.1.3
```

**先にJSR上で公開先のパッケージを作成し、設定項目でGitHubActionsが実行されるGitHubリポジトリを指定**してから、tagをプッシュしてGitHub Actionを実行します。
（そうしないとエラーが出てしまいます。）

```sh
git tag -a v0.0.0 -m "version 0.0.0"
git push origin v0.0.0
```

:::message
jsrの不具合として、readmeに表示した`Warning`内のリストが正常に表示されない問題がありそうです。
:::

## スコアを上げる

どうせ公開するならスコアが高い方が嬉しいです。現状は以下がスコアの基準になります。

### ドキュメントを書く
- exportされた関数・クラス等のシンボルにJSDocを書く
- Readmeかモジュールドキュメントを書く・例も書く

### ベストプラクティス
- slow typesを使わない
  - exportされた関数に戻り値の型を明示する
    - これは`deno lint`でも指摘してくれます

### 説明性
- JSRでDescriptionを書く

### 互換性
- Deno・Bun・npm等複数のランタイムとの互換性がある
- CI/CDワークフローから公開される

## バッジをつける

バッジをつけることで、JSRで公開されたパッケージであることを示すことができます。

```markdown
[![JSR](https://jsr.io/badges/@<scope>/<package>)](https://jsr.io/@<scope>/<package>)
```

# まとめ

chevrotainを使ってINIファイルとJSONファイルのパーサを作れました。書き心地は悪くないですが、バンドルサイズなどが気になります。検証は必要そうですね。（いつかやるかも）

とりあえずのところは自分用で使ってみて使い心地試したいですね
