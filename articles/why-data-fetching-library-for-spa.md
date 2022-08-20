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
を対象とします。また、筆者の興味がないということで、SSR や ISR はこの記事の議論では対象にしません[^ssr-isr]。読み込みパフォーマンスについても要求が控えめです。

[^ssr-isr]: 用語自体は[【Next.js】CSR,SSG,SSR,ISR があやふやな人へざっくり解説する](https://zenn.dev/akino/articles/78479998efef55) をご参照ください。

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

know if data have already been fetched and is available
know if fetching is currently in progress
know if fetching has failed
deduplicate requests
re-fetch on error
cache data and invalidate the cache
handle mutations with dependent data (think of when changing one entity affects other entities)
optimistic updates
reflect Server State in UI
コントロールできないリモートに[永続化]されている
取得・更新のための非同期のライブラリ API が必要
知らずのうちに他の人から更新される可能性がある（暗示的な[所有権]の[共有]）
注意しないと状態が消費期限切れになる
[キャッシュ]。[プログラミング]でもっとも大変かも。
同じデータへのリクエストの重複排除
[バックグラウンド]での[消費期限]切れデータの最新化
いつ[消費期限]切れになるか知ること
データ更新の反映を速く
[パフォーマンス]。例えば、[ページネーション]や[遅延読み込み]。
[Server State] のメモリと GC の管理
構造的共有による取得結果のメモ化

@TODO（列挙）

https://alexei.me/blog/state-management--separation-of-concerns/#solutions-for-server-state

https://tanstack.com/query/v4/docs/overview

これだけの課題が挙がるように、Server State に関する実装は骨の折れる仕事です。幸い、我々はすでに先人が築いた解決策としてのライブラリに頼ることができます。

## データ取得ライブラリ

データ取得ライブラリ（Data Fetching Library）は、Server State をいい感じに管理してくれるライブラリです。「取得」とだけありますが、有名なライブラリはキャッシュ機構や Hook 形式の API も備えています。

典型的には、`useQuery` のような Hook によって参照系のエンドポイントから Server State を取得します。更新系に対しても `useMutation` のような Hook によってサーバーに作用したり更新によって古くなった手元のキャッシュを無効化したりします。

### ライブラリの例

このようなライブラリとしては React Query[^tanstack-query] や SWR、RTK Query が有名です。

[^tanstack-query]: v4 で TanStack Query に改名した模様。

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

データ取得ライブラリという名前にあるように、取得する方法やそのタイミングを指定するのは実装上避けられない処理です。

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

State 管理ライブラリに備わる機能により、何らかのイベントが発火されたときに取得する方法です。例としては、[Recoil](https://recoiljs.org/) で初期値の読み取りのタイミングで非同期処理にてデータを取得できます。ニッチですが [Redux Dynamic Modules](https://redux-dynamic-modules.js.org/) の `initialActions` はモジュール読み込み時に取得するよう指定できます。

下のコードは、Recoil にて `myState` を初めて `useRecoilValue` 等で参照する際に取得します。

```js
const myState = selector({
  key: "myState",
  get: () => fetchMyState(),
});
```

十分シンプルで問題ないようにも見えますが、Server State を扱う難しさは緩和してくれません。自分で頑張るか追加のライブラリを使う必要があります。

## 利用方法

Global State 全般に言えることですが、取得した State をどうコンポーネントで使うかという方法も色々やり方があります。いくつかの方法では、そのコンポーネント内で閉じてよい State も外に露出してしまう欠点があります。

### バケツリレー

上の階層で保持する State を Props で子に受け渡してくる方法です。いわゆる「バケツリレー」です。

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

管理が大変。何もしないとレンダーが何回も走ってダメ。Constate 使ったり。

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

使うだけなら良さそうだが、Server State を扱うための変数や State がボイラープレート的なのは変わらない。Loading, Succeeded, Failed をそれぞれ用意とか[^caveat-for-recoil]。

[^caveat-for-recoil]: Recoil などで Suspense や ErrorBoundary を積極利用する場合は大丈夫かもしれない。

# データ取得ライブラリの利点

データ取得ライブラリを使う利点を紹介し、従来の課題がどのように解決または緩和されるか説明します。

また、これによって良い設計につながることも示します。

複雑で誤解しやすいコードをたくさん削除でき、わずかな [React Query] のコードに置き換えられる。
メンテナンス性が上がり、新しい [Server State] とつなげる心配をせず新しい機能を実装できる。
アプリケーションが速く動き反応も良いとユーザーに感じさせる。直接的にインパクトがある。
ネットワークのバンド幅を抑えたりメモリのパフォーマンスを向上させる。
たった 1 行のコードで、プロジェクト内のデータ取得のロジックを単純化し、さらにこれらの素晴らしい機能をすぐに利用できるようになります：

速い、 軽量 そして 再利用可能 なデータの取得
組み込みの キャッシュ とリクエストの重複排除
リアルタイム な体験
トランスポートとプロトコルにとらわれない
SSR / ISR / SSG support
TypeScript 対応
React Native
高速なページナビゲーション
定期的にポーリングする
データの依存関係
フォーカス時の再検証
ネットワーク回復時の再検証
ローカルキャッシュの更新（Optimistic UI）
スマートなエラーの再試行
ページネーションとスクロールポジションの回復
React Suspense
[RTK Query] は強力なデータ取得と[キャッシュ]ツール。[Web App] によくあるケースを簡単にし、手書きのデータ取得とキャッシュ処理を自分で書く必要をなくすよう設計されている。
[Apollo Client] は読み込みとエラーの状態の追跡を含めて[リクエスト]サイクルを最初から最後まで対応する。最初の[リクエスト]まで[ミドルウェア]や[ボイラープレートコード]は必要なく、レスポンスの変換やキャッシュを心配しなくてよい。コンポーネントが必要なデータを説明し、[Apollo Client] に仕事をさせるだけでいい。
`useQuery` Hook は [React.Hook] をクエリをコンポーネントと繋ぎコンポーネントが結果を即時にレンダーできるようにする。`useQuery` はデータ取得処理を隠蔽し、読み込みとエラーの状態を追跡し、ＵＩ をアップデートする。この隠蔽は、クエリ結果をプレゼンテーショナルコンポーネントにつなげるのをスムーズにする。
（略）
[Apollo Client] に切り替えるすると、データ管理に関するコードのほとんどが減らせることがわかるでしょう。
[Apollo Client] でわずかなコードを書いていたとしても、機能の妥協は意味しない。`useQuery` は楽観的 UI や再読み込みやページネーションなど先進的な機能をサポートする。
[Relay] はどの大きさでも高いパフォーマンスを発揮するよう作られている。[Relay] はコンポーネントの数が 10 でも １００ でも １０００ でもデータ取得の管理をを簡単にし続ける。[インクリメンタルコンパイル]により、[App] が成長してもイテレーション速度を保てる。
[* イテレーションを高速に保つ]
[Relay] はデータ取得を[宣言的]にする。コンポーネントは、データの依存を宣言すればよく、どうやって取得するかは心配いらない。[Relay] はそれぞれのコンポーネントが必要とするデータはフェッチされていて利用可能だということを保障する。これは、コンポーネントを切り離し再利用を促進する。
[Relay] を使えば、コンポーネントとそのデータ依存は他の部分を変えることなく高速に書き換えられる。リファクタリングや [App] への変更で意図せず他のコンポーネントを壊さないことを意味する。

## 宣言的に書ける

データ取得ライブラリを使うと、データ取得を簡単に**宣言的**（Declarative）に書けます。React 等の利点として良く出てくる「宣言的 UI」の「宣言的」の意味と同じです。

従来の命令的なデータ取得処理では、「マウントされたら、もしくは、イベントが発火したら、取得処理を実行し、State を更新する。」と書くことで要求を実現します。これでも良いようにも見えますが、「なんでもいいからこういうデータをいい感じに欲しい」という要求に対しては余計なことを考慮しなければならないと考えることもできます。

```js
const [myState, setMyState] = useState();
useEffect(() => {
  async function fn() {
    const serverState = await fetchServerState(); // 2. 取得処理を実行し
    setMyState(serverState); // 3. State を更新する
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
親で子の読み込み状態を考慮して親で表示を切り替えるのは従来は面倒くさかった。それが今では宣言的に書ける。
Suspense により読み込み中のちぐはぐさは回避できるようになった。その分疎結合に。

中央で管理するのが Suspense と相性が悪いから分散して「末端」取得するのでなく、分散統治するために「末端」取得と Suspense を利用するという、「発想の転換」。

### コンポーネント内の見通しが良くなる

親や子となるコンポーネントもあまり意識せずに済むため、コンポーネント内の見通しが良くなります。

@TODO

責務に集中できるということ。Pages なら Pages 内が、Organisms なら Organisms 内が。

State は必要な場所だけで考えれば良い。

Organisms 内に置かれるので。取得と利用とで距離が近づく。

不必要な State やボイラープレートが減る。

# 新しく生まれる課題や議論

データ取得ライブラリを導入することで発生する課題や議論（の中で筆者が見つけられたもの）を挙げます。新しい技術には何かしらの欠点はありますので、実戦投入時にはトレードオフを考えどこを落とし所にするかという話になります。その検討の参考になればと思います。[^recoil]

[^recoil]: 公開時点では、筆者は Recoil のようなライブラリを基盤にしたデータ取得ライブラリにより、これらの課題が緩和されるのではないかと考えている。ただし、それをやっている形跡が検索にヒットしないため、的を外している可能性がある。

## クエリキーという概念

データ取得ライブラリを導入することでコードに混乱を招く可能性が高い概念は、おそらくはクエリキーとその管理ではないかと思います。

@TODO

https://tanstack.com/query/v4/docs/guides/query-keys

クエリキーとは。

キー管理。

## エンドポイント間の依存管理

キャッシュに関連して、ある Mutation ではどのキャッシュを無効化すべきか、という考慮事項が発生します。クエリキーの管理の議論の中の、エンドポイント間の依存をどう管理してどうコードに確実に記載するかです。

@TODO

ちょっとした画面ではキーとフェッチ関数を毎回書けばやっていけるかもしれないけど。筆者の興味としては中規模以上の SPA なので、そうそう破綻すると思う。

GraphQL で解決するケースもある。

レイヤーを作って、すべて Custom Hook に封じ込めるやりかたもあるらしい。（ボイラープレート増えてるじゃあないか）

## テストどうする？

データ取得ライブラリによって、少なくともテストの基盤的にコードには影響がでます。データ取得ライブラリは自律的で高度な機能を備えており、かつ、それを使うコードがテスト対象になるためです。

@TODO

https://tanstack.com/query/v4/docs/guides/testing

とは言ってもデータ取得ライブラリ固有の対処自体は簡単。「バカ」にする設定を入れた Client を注入する。詳しくは書くライブラリの説明を参照のこと。

データをとってくるという観点のテストが大変。MSW 等を使うと良い。

organisms のテストとかカタログが云々というのは、それに加えて react-testing-library とか Storybook interaction addon とかで解決。

この辺はすでに識者が記事を書かれているので、検索されたし。

## 依存が増える

新しいライブラリを導入すると（当然）バンドルサイズは増えます。フロントエンドの宿命として、アプリケーションのバンドルサイズは常に懸念事項です。

@TODO

react と react-dom 18.2.0 では、計 137 kB、gzip 圧縮後で 計 45 kB。それと比較して…

アプリケーションのサービス戦略的に読み込み速度が重要だとか react-query や Apollo 以上のカスタマイズ性が必要とかで無いなら、税金だと思ってバンドルサイズを増やして開発スピードを上げたほうがいいんじゃない？

## Global State との分断

多くのデータ取得ライブラリは、Server State のみを扱うことを念頭に作られています。Client State のことはあまり考慮されていないので、 Global State が Client State と Server State に分断されてしまいます。具体例としては、React-Query にある Server State をどう Redux で管理する Client State に影響させるかということです。

@TODO

やりようはあるので、これが問題となるのは Client State が多く、かつ、Server State と連携が強いアプリケーション。（筆者は興味があるのでわざわざ取り上げている。）

この記事の "React-Query — The pitfalls" は、React-Query （や他のライブラリ）かなり

https://medium.com/duda/what-i-learned-from-react-query-and-why-i-will-not-use-it-in-my-next-project-a459f3e91887#ccd3

Server State やデータ取得課題を解決するには大きすぎるという指摘。

分断を避ける解決策の例としては [Recoil Relay](https://recoiljs.org/docs/recoil-relay/introduction) がある。
