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

## React の State について

この記事では、React の State を以下の記事のように分類して考えています。

https://zenn.dev/akfm/articles/react-state-scope

- Local State: `useState` などによって管理するような State
- Global State: 複数のコンポーネントから利用されうる、もしくは、ページを跨いで利用する State。
  - Client State: UI の状態など
  - Server State: （すぐ下で説明）

### Server State

Server State は、API サーバーからのレスポンスやそのキャッシュのことです。Redux や Recoil などの State 管理ライブラリで管理されたり後述のデータ取得ライブラリで管理されたりします。

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

データ取得ライブラリの導入以前ではどうやっていたのかを見ていきましょう。データに関しての取得の軸と利用の軸があると考えています。

## 取得方法

データ取得ライブラリという名前にあるように、取得する方法やタイミングは実装上避けられない処理です。

### マウント時

`useEffect` や `componentDidMount` を使い、コンポーネントがマウントされたときに取得する方法です。

```js
const [myState, setMyState] = useState();
useEffect(() => {
  async function fn() {
    setMyState(await fetchMyState());
  }
  fn();
});
```

@TODO

使ってはいけないとアナウンス
https://zenn.dev/dai_shi/articles/dee54f995e6e74
deps 管理が難しくで読み込みが不要だが起きたり必要だが起きなかったりする。
命令的コード

### State 管理ライブラリのイベントで

State 管理ライブラリに備わる機能により、何らかのイベントが発火されたときに取得する方法です。例としては、[Recoil](https://recoiljs.org/) で初期値の読み取りのタイミングで非同期処理にてデータを取得できます。ニッチですが [Redux Dynamic Modules](https://redux-dynamic-modules.js.org/) の `initialActions` はモジュール読み込み時に取得するよう指定できます。

下のコードは、Recoil にて `myState` を初めて `useRecoilValue` 等で参照する際に取得します。

```js
const myState = atom({
  key: "myState",
  default: selector({
    key: "myState/Default",
    get: () => fetchMyState(),
  }),
});
```

問題ないようにも見えますが、Server State の難しさは緩和してくれません。自分で頑張る必要があります。

## 利用方法

取得した State をどうコンポーネントで使うかという方法も色々やり方があります。いくつかの方法では、そのコンポーネント内で閉じてよい State も外に露出してしまう欠点があります。

### バケツリレー

上の階層で保持する State を Props で子に受け渡してくる（バケツリレー）方法です。

```jsx
const Page = () => {
  const [myState, setMyState] = useState();

  // 何らかの方法で myState の値を取得

  // 実際は使う場所まで何回も繰り返し渡す
  return <Reaf myState={myState} />;
};

const Reaf = ({ myState }) => <span>{myState}</span>;
```

@TODO

距離があってダメ。ボイラープレート的でコードが膨れて駄目。

### Context

上の階層で保持する State を Context API を使って参照する方法です。

```jsx
// Context の準備は省略

const Page = () => {
  const [myState, setMyState] = useState();

  // 何らかの方法で myState の値を取得

  return (
    <MyStateProvider value={myState}>
      <Reaf />
    </MyStateProvider>
  );
};

const Reaf = ({ myState }) => {
  const myState = useMyState();
  return <span>{myState}</span>;
};
```

@TODO

管理が大変。何もしないとレンダーが何回も走ってダメ。

### 従来のデータ管理ライブラリ

Global State を扱おうとするデータ管理ライブラリに格納しておいて、それを参照する方法です。

```jsx
const Page = () => {
  const [myState, setMyState] = useState();

  // 何らかの方法で myState の値を取得

  return <Reaf />;
};

const Reaf = ({ myState }) => {
  const myState = useMyState((state) => state.myState);
  return <span>{myState}</span>;
};
```

@TODO

使うだけなら良さそうだが、Server State を扱うための状態がボイラープレート的なのは変わらない。Loading, Succeeded, Failed をそれぞれ用意とか[^caveat-for-recoil]。

[^caveat-for-recoil]: Recoil などで Suspense や ErrorBoundary を積極利用する場合は大丈夫かもしれない。

# データ取得ライブラリの利点

@TODO

## 宣言的に書ける

@TODO

## Server State の管理が楽になる

@TODO

ボイラープレートの削減。"画面" 遷移。非戦略的動作の外部調達。
打ちっぱなし能力。宣言的に書ける。
Loading, Succeeded, Failed も折込済。

## `useEffect` を回避できる

@TODO

## Suspense と相性がいい

@TODO

相乗効果

読み込みの発火（Data-Fetching → Suspense）も、統合も（Suspense → Data-Fetching）。ErrorBoundary も？

## パフォーマンスが良くなる

@TODO

「リアクティブ性」によりレンダリングの無駄が少ない。しかも自分でやらなくてよい。

## 分割統治が可能になる

@TODO 内部で分割したほうが良いのかも

あまり説明や解説には出てこないけど。おまけにしてはかなり重要な観点だと思っている。

他の部分の無視・設計上の利点。
Relay の "Keeps iteration quick" の説明で少し触れられている。
ある状態は、必要な場所だけで考えれば良い。取得と利用の距離が近づく。

どのコンポーネント層で通信するか？
しかしこれは Pages か Organisms かで決まっている。書かなくていいかも。
Atomic Design でいうところの Organism 単位で読み込みして問題ない。複雑さが減る。Pages から下ろして来ないで済む。

cache が大体はいい感じにやってくれるので富豪的に書ける。

Suspense により読み込み中のちぐはぐさは回避できるようになった。

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

Next.js とか React Location とか React Router v6.4 で解決するやつ。Recoil には [pre-fetching](https://recoiljs.org/docs/guides/asynchronous-data-queries/#pre-fetching) の挙動もできる。

## テストどうする？

organisms のテストとかカタログが云々というのは msw とか storybook interaction addon とかで解決。

## Global State との分断

Global State 内の一貫性というか、Client State と Server State のシームレスな連携は課題。Recoil とかとつなげるの。

https://medium.com/duda/what-i-learned-from-react-query-and-why-i-will-not-use-it-in-my-next-project-a459f3e91887
