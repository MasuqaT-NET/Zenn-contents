---
title: "データ取得ライブラリを SPA に導入するとなぜ嬉しいのか"
emoji: "🧱"
type: "tech"
topics: ["frontend", "spa", "react", "reactquery"]
published: false
---

# TL;DR

@TODO

（データ取得ライブラリは Server State の管理を簡単にします。その延長で ～ の利点を得られます。）

# スコープ

この記事は React による Client Side Rendering(CSR) の SPA を対象とします。おそらく利点は React に限りませんが筆者が慣れている React を対象とします。また、筆者の興味がないということで、SSR や ISR はこの記事の議論では対象にしません[^ssr-isr]。

[^ssr-isr]: 用語自体は[【Next.js】CSR,SSG,SSR,ISR があやふやな人へざっくり解説する](https://zenn.dev/akino/articles/78479998efef55)をご参照ください。

# 予備知識

## Server State

Server State は、API サーバーからのレスポンスやそのキャッシュのことです。Redux や Recoil などの State 管理ライブラリで管理されたり後述のデータ取得ライブラリで管理されたりします。この概念は下記の記事が非常に参考になります。

https://zenn.dev/akfm/articles/react-state-scope

名前が違う場合もありますが、後述するライブラリの説明でも出てくる概念です。

### Server State の難しさ

Server State を扱うのは、以下の理由で難しいです。

@TODO（列挙）

https://alexei.me/blog/state-management--separation-of-concerns/

https://tanstack.com/query/v4/docs/overview

これだけの課題が挙がるように、Server State に関する実装は骨の折れる仕事です。幸い、我々はすでに先人が築いた解決策としてのライブラリに頼ることができます。

## データ取得ライブラリ

データ取得ライブラリ（Data Fetching Library）は、Server State をいい感じに管理してくれるライブラリです。「取得」とだけありますが、有名なライブラリはキャッシュ機構や Hook 形式の API も備えています。

### ライブラリの例

このようなライブラリとしては React Query[^tanstack-query] や SWR、RTK Query が有名です。

[^tanstack-query]: v4 で TanStack Query に改名？

https://tanstack.com/query/v4/

https://swr.vercel.app/ja

https://redux-toolkit.js.org/rtk-query/overview

GraphQL クライアントである Apollo Client や Relay、urql も同じような機能を備えているため、データ取得ライブラリという括りに入ります。

https://www.apollographql.com/docs/react/

https://relay.dev/

https://formidable.com/open-source/urql/

### HTTP クライアントとの違い

このデータ取得ライブラリは従来からある HTTP クライアント（Axios や Fetch API）とは違います。

@TODO

cache がある。React 等の UI ライブラリを念頭においていて、 Hook として提供されている。キャッシュと「リアクティブ性」がミソ。

# 従来のやりかた

@TODO 構成を見直す（状態管理と `useEffect` が両立しているので。落穂ひろいも中途半端）

## 管理方法

@TODO

### ページコンポーネント＋バケツリレー

@TODO

距離があってダメ。ボイラープレート的でコードが膨れて駄目。

### Context

@TODO

管理が大変。レンダーが何回も走ってダメ。

### 従来のデータ管理ライブラリ

@TODO

（Redux うんぬんは React Query の説明にあるとおりに書く）

### （共通の問題）

@TODO

ページ間で明示的に共有するとページ内で閉じてよいものが無駄にグローバルに露出する。

## 取得方法

@TODO

### `useEffect` や `componentDidMount`

@TODO

使ってはいけないとアナウンス
deps 管理が難しくで読み込みが不要だが起きたり必要だが起きなかったりする。
命令的コード

### ステート管理ライブラリのイベント

@TODO

# 利点

@TODO

## 宣言的に書ける

@TODO

## Server State の管理が楽になる

@TODO

ボイラープレートの削減。"画面" 遷移。非戦略的動作の外部調達。
打ちっぱなし能力。宣言的に書ける。

## Suspense と相性がいい

@TODO

相乗効果

読み込みの発火（Data-Fetching -> Suspense）も、統合も（Suspense -> Data-Fetching）。ErrorBoundary も？

## `useEffect` を回避できる

@TODO

https://zenn.dev/dai_shi/articles/dee54f995e6e74

## 分割統治が可能になる

@TODO 内部で分割したほうが良いのかも

他の部分の無視・設計上の利点。あまり説明や解説には出てこないけど。おまけにしてはかなり重要な観点だと思っている。

「リアクティブ性」によりレンダリングの無駄が少ない。

どのコンポーネント層で通信するか？
しかしこれは Pages か Organisms かで決まっている。書かなくていいかも。
Atomic Design でいうところの Organism 単位で読み込みして問題ない。複雑さが減る。

cache が大体はいい感じにやってくれるので富豪的に書ける。

Suspense により読み込み中のちぐはぐさは回避できるようになった。

Relay の "Keeps iteration quick" の説明で少し触れられている。

# 新しく生まれる課題や議論

@TODO

## エンドポイント間の関係性の管理

@TODO

キャッシュキー。

キー管理

GraphQL で解決するケースもある。

## 依存が増える

@TODO

アプリケーションの戦略的に読み込み速度が最重要だとか react-query や Apollo 以上のカスタマイズ性が必要とかで無いなら、税金だと思ってバンドルサイズを増やして開発スピードを上げたほうがいいんじゃない？

## prefetch?

@TODO

Next.js とか React Location とか React Router v6.4 で解決するやつ。

## テストどうする？

organisms のテストとかカタログが云々というのは msw とか storybook interaction addon とかで解決。

## Global State との分断

Global State 内の一貫性というか、Client State と Server State のシームレスな連携は課題。Recoil とかとつなげるの。
