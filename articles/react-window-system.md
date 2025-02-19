---
title: "Reactでウィンドウシステムを作ってパッケージとして配布してみた"
emoji: "🪟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "フロントエンド", "bun", "npm", "ウィンドウシステム"]
published: true
---

## TL;DR

自作のフロントエンドSPAが複雑になってきて、「ウィンドウシステムほしいかもな～」ってなったので、作ってみました。

@[codesandbox](https://codesandbox.io/embed/p75t2w?view=preview&module=%2Fsrc%2Fapp.tsx&hidenavigation=1)

https://github.com/GunseiKPaseri/react-window-system

npm公開中です。

https://www.npmjs.com/package/react-window-system

## 実装周りの話

Windowsのように複数の画面を重ねながら何枚も表示できるようなUIシステムを実装しました。

### ウィンドウの部分

グラフィカルに作ろうとしたら一番面倒くさいだろう重ね合わせとかの部分は、ありがたいことに全部ブラウザがやってくれます。

ウィンドウの重ね合わせ表現はCSSの`z-index`を利用して、ユーザが選択した順に並べることで実現しています。

### ウィンドウの移動・変形

このメンドクサそうな部分は全部react-rndに委ねました。

https://github.com/bokuweb/react-rnd

これを使うと変形・移動可能な要素を作ることができます。楽だね。

### ウィンドウ管理

ここを実装しました。

と言ってもflexboxやらgridやらを使ったウィンドウの簡単なデザインと最大化・最小化・閉じる・タブあたりの状態を管理しているだけです。

### Compound Components

ウィンドウのデザインはできる限り自由にさせたいと考えています。
カスタムなウィンドウの各要素はこちらが用意したコンポーネントの組み合わせの形で、細部は自由に作れるようにしました。
最大化ボタン・最小化ボタン等は並べ替えもできます。

```tsx
export const DefaultWindow = (props: WindowUIProps) => {
  const { window, ...WindowAttr } = props;
  return (
    <Window
      {...WindowAttr}
      window={window}
      style={{
        position: "absolute",
        zIndex: window.layerIndex,
        border: "1px solid #000",
        borderRadius: "4px",
      }}
    >
      <Window.Header
        style={{
          backgroundColor: window.isActive ? "#99f" : "#ccf",
        }}
      >
        <Window.Header.Title
          style={{
            paddingLeft: 4,
          }}
        >
          {window.header}
        </Window.Header.Title>
        <Window.Header.MinimizeButton>_</Window.Header.MinimizeButton>
        <Window.Header.MaximizeButton>
          {window.maximize ? "❒" : "□"}
        </Window.Header.MaximizeButton>
        <Window.Header.CloseButton>×</Window.Header.CloseButton>
      </Window.Header>
      <Window.Body
        style={{
          backgroundColor: "#fff",
        }}
      >
        {window.body}
      </Window.Body>
    </Window>
  );
};
```

### スナップ

Windowsのスナップ機能に相当するものを実装しました。
画面を左右にぶつけると対応する画面の大きさになります。

画面端にぶつけた際表示されるパネルはCSSの[`backdrop-filter`](https://developer.mozilla.org/ja/docs/Web/CSS/backdrop-filter)を使いました。
透過度を`opacity`ではなく背景色（`rgb(0,0,0,.3)`）で指定してあげなければならない点が注意点です。アニメーションはCSSの`transition`を入れただけです。ビバCSS！

## 実装以外の話

### Bunの利用

やれpnpmだのyarnだのnpmだのというのはもういいです。時代はBunです。
とはいえ、Bunでnpmパッケージは作れるのでしょうか？できました。
`npm publish`に相当するものは無いので、公開するときだけ頑張る必要がありますが、それ以外はBunで十分でした。インストールめっちゃ早いです。

`bun.lockb`のGit管理どうするの問題は、以下の案3を採用しました。

https://zenn.dev/watany/articles/e21a54cf3d56d8

他に発生した問題としては、`npm pack`で生成したtgzを`bun add hoge.tgz`しようとするとエラーが出てしまったのが残念ですね。

```sh
$ bun add react-window-system-0.0.1.tgz
bun add v1.0.25 (a8ff7be6)
error: Package "react-dom@18.2.0" has a dependency loop
  Resolution: "vite@5.0.12"
  Dependency: "dow-systemreact-dom@https://registry.npmjs.org/re"

error: DependencyLoop

----- bun meta -----
Bun v1.0.25 (a8ff7be6) Linux x64 #1 SMP Sat Aug 19 00:27:37 JST 2023
AddCommand: extracted_packages
Elapsed: 606ms | User: 613ms | Sys: 10ms
RSS: 40.91MB | Peak: 44.33MB | Commit: 40.91MB | Faults: 0
----- bun meta -----

0   0x55808696dddb
1   ???
2   ???
3   ???
4   ???
5   ???
6   ???

Search GitHub issues https://bun.sh/issues or ask for #help in https://bun.sh/discord
```

`npm install hoge.tgz`では問題なく動き、publishした後の`bun add react-window-system`も問題なく動いたので、設定ミスではなさそうです。

https://github.com/oven-sh/bun/issues/5789

イシューが立っているので、これが解決されるのを待ちたいですね。

### package.json周り

これはReactのパッケージを公開するうえで一般的な話ですが、`react`と`react-dom`はインストール先のものと同じものを使用してもらうので`dependencies`ではなく`peerDependencies`に設定しておきます。
これだけだとインストール時におかしくなるらしいので、`devDependencies`にも記載します。

```json
  "peerDependencies": {
    "react": ">=16.3.0",
    "react-dom": ">=16.3.0"
  },
```

一方で、`react-rnd`は自分だけが使いたいバージョンを選べよいので`dependencies`に記載しています。この外側で他のバージョンが動いていても問題にはなりません。

```json
  "dependencies": {
    "react-rnd": "^10.4.1"
  },
```

……という理解なんですけど、実際に問題無いかは動かしてみないことにはわかりません。

### Tree Shaking周り

不要なコンポーネントのインポートを防ぐため、Rollupの`output.preserveModules`を`true`にしてファイル分割して提供しています。
これを有効にするとumd向けのエクスポートに失敗するので、現在cjsとesmのみ同梱してます。~~`cjs`の方は動作確認してないんだけど~~

esmだけでよかったりしませんかね？さすがにまだ早いのかな？

### CI/CD周り

CI/CDはGitHub Actionsを使っています。
CIはBunだけでよかったのですが、CDの方はnpmが無いと駄目そうなのでBunとNode.jsの両方を使っています。
リリース時に直近の`git tag`からバージョンを抜き出して、それを適用するように`npm publish`するようにしています。
package.jsonのバージョンは触らないという究極実装。

## 課題

- デザインの追加・洗練
今はWindowsっぽいものを実装していますが、Macっぽいものとかも作りたいです。仕組み的にはレイアウトさえがんばれば作れるんじゃないかと。
TreeShakingな感じで使いたいデザインだけを読み込めるようにしたいですね。

- アクセシビリティ・キーバインド
アクティブなウィンドウの切り替えや最大化・最小化などの操作をキーボードで行えるようにしたいですね。でもキージャックとかするといろいろ厄介なことになりそう……

## 最後に

オープンソースちゃんとやったことないですが、issueとかプルリクとか歓迎です。
改良やらバグ探しやら、デザイン案の追加とかしてくれると嬉しいです。
