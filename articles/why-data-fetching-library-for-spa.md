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

この記事は React による Client Side Rendering(CSR) の SPA を対象とします。おそらく利点は React に限りませんが筆者が慣れている React
を対象とします。また、筆者の興味がないということで、SSR や ISR はこの記事の議論では対象にしません[^ssr-isr]。

[^ssr-isr]:
    用語自体は[【Next.js】CSR,SSG,SSR,ISR があやふやな人へざっくり解説する](https://zenn.dev/akino/articles/78479998efef55)
    をご参照ください。

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

データ取得ライブラリ（Data Fetching Library）は、Server State をいい感じに管理してくれるライブラリです。「取得」とだけありますが、有名なライブラリはキャッシュ機構や Hook 形式の
API も備えています。

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

データ取得ライブラリの導入以前ではどうやっていたのかを見ていきましょう。データに関しての取得の観点と利用の観点があると考えています。

## 取得方法

データ取得ライブラリという名前にあるように、取得する方法やタイミングは実装上避けられない処理です。

### マウント時

`useEffect` や `componentDidMount` を使い、コンポーネントがマウントされたときに取得する方法です。

```js
const [myState, setMyState] = useState();
useEffect(() => {
  async function fn() {
    const serverState = await fetchServerState();
    setMyState(serverState);
  }

  fn();
}, []);
```

@TODO

使ってはいけないとアナウンス
https://zenn.dev/dai_shi/articles/dee54f995e6e74
deps 管理が難しくで読み込みが不要だが起きたり必要だが起きなかったりする。
命令的コード

### State 管理ライブラリのイベントで

State 管理ライブラリに備わる機能により、何らかのイベントが発火されたときに取得する方法です。例としては、[Recoil](https://recoiljs.org/)
で初期値の読み取りのタイミングで非同期処理にてデータを取得できます。ニッチですが [Redux Dynamic Modules](https://redux-dynamic-modules.js.org/)
の `initialActions` はモジュール読み込み時に取得するよう指定できます。

下のコードは、Recoil にて `myState` を初めて `useRecoilValue` 等で参照する際に取得します。

```js
const myState = selector({
  key: "myState",
  get: () => fetchMyState(),
});
```

十分シンプルで問題ないようにも見えますが、Server State を扱う難しさは緩和してくれません。自分で頑張るか追加のライブラリを使う必要があります。

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

データ取得ライブラリを使う利点を紹介し、従来の課題がどのように解決または緩和されるか説明します。

また、これによって設計上の利点につながることも示します。

## 宣言的に書ける

データ取得ライブラリを使うと、データ取得を簡単に**宣言的**（Declarative）に書けます。React 等の利点として良く出てくる「宣言的 UI」の「宣言的」の意味と同じです。

従来の命令的なデータ取得処理では、「マウントされたら、もしくは、イベントが発火したら、取得処理を実行し、状態を更新する。」と書くことで要求を実現します。これでも良いようにも見えますが、「なんでもいいからこういうデータをいい感じに欲しい」という要求に対しては余計なことを考慮しなければならないと考えることもできます。

```js
const [myState, setMyState] = useState();
useEffect(() => {
  async function fn() {
    const serverState = await fetchServerState(); // 2. 取得処理を実行し
    setMyState(serverState); // 3. 状態を更新する
  }

  fn();
}, []); // 1. マウントされたら
```

データ取得ライブラリで宣言的に書くことによって、欲しいデータのことだけ考えればよくなります。余計なことや不必要に細かいことを意識する必要はありません[^query-key][^leaky-abstraction]。こうした*重要でないこと*はライブラリに任せましょう。

```js
const { data: myState } = useQuery(["ServerState"], () => fetchServerState());
```

[^query-key]: クエリキーという概念が別に追加されているのには注意。
[^leaky-abstraction]: [漏れのある抽象化の法則（Leaky abstraction）](https://zenn.dev/nanagi/articles/0e899711611630#%E6%BC%8F%E3%82%8C%E3%81%AE%E3%81%82%E3%82%8B%E6%8A%BD%E8%B1%A1%E5%8C%96%E3%81%AE%E6%B3%95%E5%89%87%EF%BC%88leaky-abstraction%EF%BC%89)はあるため、完全には無視できないことには注意。

### Server State の管理が楽になる

宣言的に書くことで、難しい Server State の管理が楽になります。

@TODO

上記の難しさ。

キャッシュそれに対する。ボイラープレートの削減。
Loading, Succeeded, Failed も折込済。

### `useEffect` を回避できる

宣言的に書くことで、結果的に `useEffect` を使う必要が無くなります。

@TODO

## ユーザビリティを向上させやすい

データ取得ライブラリを使うと、ユーザビリティを向上させやすくなります。 ユーザビリティを向上させるために様々なテクニックの一部を簡単に実践できます。例えば、パフォーマンスを良くしたり読み込み表示を工夫したりするものです。

### パフォーマンスが良くなる

重複したデータ取得や不要な再レンダリングを防ぐため、パフォーマンスが良くなります。

@TODO

「リアクティブ性」によりレンダリングの無駄が少ない。抑えられる。しかも自分でやらなくてよい。

### 読み込み表示を工夫しやすい

React Suspense と相性が良いため、読み込み中の表示を工夫しやすいです。

@TODO

これはデータ取得に限らず Suspense 全般に言えることだが、読み込み中という状態やその境界を扱いやすい。方々でデータを読み込んでいるが変な読み込み表示にならずに済む。それにより良い読み込み表示を実現しやすい。

## コンポーネント設計が良くなる

これら 2 つの利点とその理由を活用することで分割統治が促進され、コンポーネント設計が良くなります。

この利点はライブラリの説明や有志による解説にはあまり取り上げられないように見受けられます。筆者としては、SPA 設計に影響を及ぼす重要な観点だと思っていますので強調します。

### コンポーネント間がより疎結合になる

分割統治が進むことでコンポーネント間のデータの受け渡しや協調処理が減り、コンポーネント間がより疎結合になります。

@TODO

他の部分の無視・設計上の利点。
Relay の "Keeps iteration quick" の説明で少し触れられている。

レイヤーを分けたときにどのコンポーネント層で通信するか？という論点がある。Atomic Design の粒度において、Pages と Organisms の分類を層として取り上げる。

従来は Page が統括的に読み込んでくださいとなるので Pages にたくさんのデータ取得処理が書かれる。バケツリレーの「暴力」も発生するかも。Organisms で呼ぶなら、他 Organisms など外部との協調処理がある。競合が起きないようにするとか。

データ取得ライブラリを使うと、Organisms 単位で読み込みして問題ない。キャッシュが大体はいい感じにやってくれるので富豪的に書ける。Pages や Templates は～から解放されて別の責務を、Organisms はより「有機的」に。

Server State を弄る際、他のコンポーネントから使われることを意識しなくてよい。

Suspense と相性がいいので。
相乗効果があると筆者は考えている。読み込みの発火（Data-Fetching → Suspense）も、見た目の統合も（Suspense → Data-Fetching）。
親で読み込み状態を切り替えるのは従来は面倒くさかった。宣言的に。
Suspense により読み込み中のちぐはぐさは回避できるようになった。その分疎結合に

### コンポーネント内の見通しが良くなる

親や子となるコンポーネントもあまり意識せずに済むため、コンポーネント内の見通しが良くなります。

@TODO

責務に集中できるということ。Pages なら Pages 内が、Organisms なら Organisms 内が。

状態は必要な場所だけで考えれば良い。

Organisms 内に置かれるので。取得と利用とで距離が近づく。

不必要な状態やボイラープレートが減る。

# 新しく生まれる課題や議論

@TODO

## クエリキーという概念

@TODO

キー管理

## エンドポイント間の関係性の管理

@TODO

GraphQL で解決するケースもある。

## 依存が増える

@TODO

アプリケーションの戦略的に読み込み速度が最重要だとか react-query や Apollo 以上のカスタマイズ性が必要とかで無いなら、税金だと思ってバンドルサイズを増やして開発スピードを上げたほうがいいんじゃない？

## テストどうする？

organisms のテストとかカタログが云々というのは msw とか storybook interaction addon とかで解決。

## Global State との分断

Global State 内の一貫性というか、Client State と Server State のシームレスな連携は課題。Recoil とかとつなげるの。

https://medium.com/duda/what-i-learned-from-react-query-and-why-i-will-not-use-it-in-my-next-project-a459f3e91887
