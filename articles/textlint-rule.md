---
title: "Textlintのルールを色々作ってみた"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["textlint"]
published: true
---
## Textlint、いいぞ

Textlintはeslintや日本語に静的チェック・自動修正をかけることができるツールです。

例えば、特定の環境依存文字を使わないようにする、助詞の重複の検出……

これを適用して技術文章を書いても良し、小説を書いても良し。

というわけで、手元でいじくってみたのですが、なんかこう……もっとガツガツ指摘してほしいなと思ったので、いい感じのルールができたので紹介します。

## 1. 漢字の開閉(textlint-rule-ja-tojihiraki)

漢字の開閉を統一するルールです。

<!-- textlint-disable -->

例えば「感じること」「感じる事」、どちらが良いでしょうか。色々考えあると思いますが、これを統一するルールです。
漢字を開いた方が分かりやすい、ということで開くルールについては沢山用意されています。

https://www.npmjs.com/package/textlint-rule-ja-hiragana-hojodoushi
https://www.npmjs.com/package/textlint-rule-ja-hiragana-fukushi
https://www.npmjs.com/package/textlint-rule-ja-hiragana-keishikimeishi
https://www.npmjs.com/package/textlint-rule-ja-hiragana-daimeishi

でも、何でもかんでも開いてしまうと却って平仮名ばかりで読みにくくないですか？　私は読みにくいです。（この辺では積極的に閉じてみています）
だから、*漢字を閉じることも強制できる*ルールを作りました。

既に形態素ごとにルールが作成されていますが、形態素単位で開くか閉じるかって決めにくいと思うんです。
例えば副詞の「極めて」は閉じていて欲しいけど「漸く」は開いていて欲しい、とか。

<!-- textlint-enable -->

1つにまとまっているものがあったので、これをベースに魔改造しました。

https://github.com/akiomik/textlint-rule-ja-hiraku

出来上がったものがこちらです。

https://www.npmjs.com/package/textlint-rule-ja-tojihiraki

この人がすでに「開いたほうがいいもの」を選んでくれたので、これをrecommendとして「無視」「開く」「閉じる」について自由に例外を追加したりできるようにしました。

いきなり適用すると、かなりの量のエラーが出ます。
正直大分うるさいし、形態素解析がおかしかったりするので、ガンガン自分好みに例外を追加していくのがいいと思います。

## 2. 見逃しがちなTypo検出(textlint-rule-ja-overlooked-typo)

一方でこちらは、ぱっと見ではわからないけど明らかなタイプミスを検出する比較的どこでも使えそうなルールです。

<!-- textlint-disable -->

例えば、「ス二ーカー」、「たいヘん」のようなぱっと見ではわからないタイプミスを検出します。（「二」が漢字、「ヘ」がカタカナ）

<!-- textlint-enable -->

こんなミスしないでしょ、と思いきや意外とあるんですよね。

基本的には正規表現のごり押しで、カタカナが並ぶ中にある不自然な漢字、ひらがなを検出しています。

「データベースへアクセス」というようなケースでは引っかからないようにしています。

https://www.npmjs.com/package/textlint-rule-ja-overlooked-typo

## 開発に使える諸々

### コミュニティ

https://github.com/textlint-ja

日本語用のコミュニティがあるので、参考になる情報がたくさんあります。

評判良ければコミュニティに移動させるのもアリですね。（こういうのやったことないので、これすることで何がどうなるのかわかりませんが）

### テスト

挙動を確認するため用のデータセットがあります。

https://github.com/textlint-ja/technological-book-corpus-ja

npmでグローバルにインストールして、ディレクトリ中で以下のようなコマンドを打つと、一気に確認してくれます。（`./lib`は出力されているルール）

ちなみに、出力される`./lib`のようなディレクトリの直下にある`.js`ファイルはすべてルールとして扱われるので、関係のないファイルはサブディレクトリに移動させましょう。

```sh
npm install --global technological-book-corpus-ja
technological-book-corpus-ja | xargs textlint --rulesdir ./lib -f pretty-error
```

Windowsでもできないか工夫してみましたが、WSLじゃないとうまく動いてくれ無さそうでした。

### 最後に

Textlint便利なので、Zennにも導入してみました。これを機に気の迷いで始めたピリオド文章も辞めました。
いらないルールも多かったですが「適応」の誤字などを見つけることができました。
皆もtextlintを入れて快適な執筆ライフを！
