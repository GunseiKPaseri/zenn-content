---
title: "Deno向けセッション管理ミドルウェアを実装してみた"
emoji: "🏉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["deno", "opine", "session"]
published: false
---

## TL;DR

denoを触ってたら，opine向けのセッションミドルウェアが荒れてたので~~渋々~~実装しました．

既存のものに比べ，セッションに型をつけることができる点が評価点かなと思います．

## 現状

### サーバフレームワーク

デファクトスタンダードが決まってないので荒ぶってるけどoakっていうのが流行っているらしい．

https://zenn.dev/kawarimidoll/articles/8031c2618fedca

なんでopineを選んだのかといえば，expressの知識生かして実装するつもりだったからなんですが，環境の整備っぷりがアレなので旨味も薄いかも……

### セッション管理系諸々

こっちもなんか多すぎる

- https://deno.land/x/session
- https://deno.land/x/sessions
- https://deno.land/x/session_middleware
- https://deno.land/x/opine_sessions

で，通常ならこれらを利用したいところですが，どうもイマイチで…
一番使い勝手が良さそうなのはsession_middlewareなんですが，まさかの中身の殆どがjavascriptという…

せっかくのopineでmiddlewareが使えてないのも辛い

## 作ったの

https://github.com/GunseiKPaseri/deno-session

今後のメンテはまだ考えてないのでdeno.landには上げてないです．これ以上増やしてもあれですし……

### storeとロジックの分離

多くのセッションパッケージと同様，ストレージに書き込む部分を`Store`として分離していて，それぞれの環境に合わせて書き換えることができるようになってます．

### セッションデータに型を付加

多くが`Record<string, string>`で済ませている所を，型補完できるようにしてみました．内部はjsonだからそれ自体は容易なんだよね．

### 課題

- テストが書けていない
  - superdenoのクッキー周辺がなんか怪しいので後回しに……
- express-sessionに比べるとまだまだ貧弱
- middlewareにより付加された型補完があれ
  - 都度エントリーに`req: OpineRequestWithSession<sd>`って書かなきゃいけないのがくどい…けど他の方法が浮かばず

## まとめ

型が付加できて差別化はできてはいる．

oak触ってみてダメそうなら帰って色々いじってみるか
