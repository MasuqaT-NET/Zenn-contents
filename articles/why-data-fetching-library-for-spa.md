---
title: "データ取得ライブラリを SPA に導入するとなぜ嬉しいのか"
emoji: "🧱"
type: "tech"
topics: ["frontend", "spa", "react", "reactquery"]
published: true
---

# TL;DR

TanStack Query や SWR のようなデータ取得ライブラリは、難しいとされる Server State 管理を簡単にします。ユーザビリティやコンポーネント設計の品質も向上させます。導入する際にはいくつか注意する点があります。

（かなり長くなってしまったため、目次や目に留まった箇所だけ読むのも良いかと思います）

# スコープ

この記事は Client Side Rendering(CSR) の SPA を対象とします。筆者（の業務）の関心や要求が少ないため、SSR や ISR はこの記事の議論では対象にしません[^ssr-isr]。読み込みパフォーマンスについても要求は控えめです。

[^ssr-isr]: 用語自体は[【Next.js】CSR,SSG,SSR,ISR があやふやな人へざっくり解説する](https://zenn.dev/akino/articles/78479998efef55)を参照すると良いでしょう。

利点や議論は特定の UI ライブラリ・フレームワークに限りませんが、筆者が慣れている React を使って説明します。

# 予備知識

## React の State について

この記事では、React の State を以下の記事のように分類して考えています。

https://zenn.dev/akfm/articles/react-state-scope

- Local State: `useState` などで管理するような State
- Global State: 複数のコンポーネントから利用されうる、もしくは、ページをまたいで利用される State。
  - Client State: UI の状態など
  - Server State: （すぐ下で説明）

### Server State

Server State は、API サーバーからのレスポンスデータやそのキャッシュのことです。Redux や Recoil などの State 管理ライブラリで管理されたり後述のデータ取得ライブラリで管理されたりします。

後述するライブラリの説明でも出てくる概念です。Server State とは名前が違う場合もあります。

この記事では、リモートにあるものを単に「データ」とし、手元にあるものを「Server State」とします。それらをつなぐ必要がある点で、「Server State 管理」には「Server State」だけでなく「データ」の取得や更新も含みます。

### Server State 管理の難しさ

Server State 管理の難しさは以下のとおりです[^server-state-refs]。

- 特有のライフサイクル
  - 手元からコントロールできないリモートで（取得対象の）データが永続化されている
  - 知らない間に他からデータが更新される可能性がある
  - いつ Server State を有効期限切れと見なすか
  - 有効期限切れ Server State をバックグラウンドで最新化すること
  - 注意しないと Server State が有効期限切れになること
  - 手元に置いた Server State のためのメモリと GC の管理
- 取得に関わる処理と関心の多さ
  - Server State を使えるか知ること
  - データ取得処理が進行中か知ること
  - すでにデータを取得したか知ること
  - データ取得処理に失敗したか知ること
  - エラー発生時の再取得
  - 非同期のライブラリ API が必要になること
- キャッシュ処理の難しさ
  - （そもそも）プログラミングでもっとも大変かもしれない処理
  - キャッシュ機構とキャッシュの無効化
  - 更新処理においてその更新に影響を受ける Server State の判定と扱い
  - Structural Sharing による取得結果のメモ化
- （体感を含めた）パフォーマンス
  - 同じデータへのリクエストの重複排除
  - Server State の更新を UI に速く反映すること
  - ページネーションや遅延読み込みなどのパフォーマンス最適化
  - 「楽観的な更新」の実現の大変さ

[^server-state-refs]: [State Management: Separation of Concerns | Alexey Antipov](https://alexei.me/blog/state-management--separation-of-concerns/#solutions-for-server-state) および [Overview | TanStack Query Docs](https://tanstack.com/query/v4/docs/overview#motivation) から抽出し、筆者がグループ化して日本語に訳しました。

これだけの課題が挙がるように、 Server State 管理に関する実装は骨の折れる仕事です。ボイラープレートも多くなってしまいます。幸い、我々はすでに先人が築いた解決策としてのライブラリに頼ることができます。

## データ取得ライブラリ

データ取得ライブラリ（Data Fetching Library）は、Server State 管理をいい感じにやってくれるライブラリです。名前には「取得」とだけありますが、有名なライブラリはキャッシュ機構や Hook 形式の API も備えています。

典型的には、`useQuery` のような Hook によって参照系のエンドポイントからデータを取得するか手元の Server State を利用します。更新系に対しても `useMutation` のような Hook によってサーバーに作用したり更新によって古くなった Server State を無効化したりします。

### ライブラリの例

このようなライブラリとしては TanStack Query(React Query)[^tanstack-query] や SWR、RTK Query が有名です。

[^tanstack-query]: v4 で TanStack Query に改名したようです。

https://tanstack.com/query/v4/

https://swr.vercel.app/ja

https://redux-toolkit.js.org/rtk-query/overview

GraphQL クライアントである Apollo Client や Relay、urql も同じような機能を備えています。データ取得ライブラリの一種と見なすことができると筆者は考えています。

https://www.apollographql.com/docs/react/

https://relay.dev/

https://formidable.com/open-source/urql/

### HTTP クライアントとの違い

データ取得ライブラリは、従来からある HTTP クライアント（[Axios](https://axios-http.com/) や [Fetch API](https://developer.mozilla.org/ja/docs/Web/API/Fetch_API)）とは違います。Server State の変化への反応性が一番の違いと言えます。

データ取得ライブラリは、React 等の UI ライブラリやフレームワークでの利用を念頭に置いています。ある Server State が更新されたとき、それに依存するコンポーネントだけが反応して自動で再レンダリングされます。データ取得ライブラリは組み込みでキャッシュ関連の機能が充実しており、UI に合わせて柔軟に設定できます。

HTTP クライアントは、あくまで取得処理のみであり、反応性やキャッシュ機能の実現はプラグインや State 管理ライブラリに委ねられています。

反応性とキャッシュの組み合わせにより、データ取得ライブラリと従来の HTTP クライアントとでは利用効果に大きな差が生まれます。

# 従来の Server State 管理方法

データ取得ライブラリの導入以前では Server State 管理をどのように実現しようとしていたのかを見ていきましょう。データの取得の観点と Server State の利用の観点があると考えています。

どの方法の組み合わせであっても、Server State 管理の難しさやボイラープレートの多さはあまり緩和してくれません。自分で頑張る必要があります。

## データの取得の観点

データ取得ライブラリという名前にあるように、取得する方法やそのタイミングを指定するのは実装上避けられない処理です。

### マウント時

`useEffect` や `componentDidMount` を使い、コンポーネントがマウントされたときに取得する方法です。

```js
const [myState, setMyState] = useState();
useEffect(() => {
  async function fn() {
    const state = await fetchState();
    setMyState(state);
  }

  fn();
}, []);
```

この方法は、React の（β 版の）公式ドキュメントでは否定的です。和訳された方の Zenn スクラップも紹介します。

https://beta.reactjs.org/learn/you-might-not-need-an-effect#fetching-data

https://zenn.dev/link/comments/eaca8bed69d68b

`useEffect` の deps によるリクエストの競合や不要なリクエストの発生、また、上述の Server State 管理の難しさにより、これを自分で実装するのは大変です。

### State 管理ライブラリのイベントで

State 管理ライブラリに備わる機能により、何らかのイベントが発火されたときに取得する方法です。例としては、[Recoil](https://recoiljs.org/) で初期値の読み取りのタイミングで非同期処理にてデータを取得できます。マイナーどころですが [Redux Dynamic Modules](https://redux-dynamic-modules.js.org/) の `initialActions` はモジュール読み込み時に取得するよう指定できます。

下のコードは、Recoil にて `myState` を初めて `useRecoilValue` 等で参照する際に取得します。

```js
const myState = selector({
  key: "myState",
  get: () => fetchMyState(),
});
```

十分にシンプルなコードで問題ないようにも見えますが、上述の Server State 管理の難しさは緩和してくれません。自分で頑張るか追加のライブラリやプラグインを使う必要があります。

## Server State の利用の観点

Global State 全般に言えることですが、Server State をどうコンポーネントで使うかという方法も色々あります。いくつかの方法では、そのコンポーネント内で閉じてよい State も外に露出してしまう欠点があります。

### バケツリレー

上の階層で保持する State を Props で子に受け渡してくる方法です。いつものやつです。State のソースをコンポーネント間で同一にしたい場合に必要となります。

```jsx
const Page = () => {
  const [myState, setMyState] = useState();

  // 何らかの方法で myState の値を取得

  // 実際は使う場所まで何回も繰り返し渡す
  return <Reaf myState={myState} />;
};

const Reaf = ({ myState }) => <span>{myState}</span>;
```

バケツリレー自体が悪いとは限りません。ただし、ルートレベルで保持した State を下層で使う場合にバケツリレーすると大変です。Server State では、取得したデータに加えて読み込み状態やエラー状態、更新用ハンドラがセットになることがよくあります。このセットがエンドポイントごとに用意されることで、バケツリレー先のコンポーネントの Props の項目はかなり増えます。使うコンポーネントまでの距離が長いと、その項目がどこからやってきたか確認するのが大変ですし、何よりもボイラープレートも増えて大変です。

### Context

上の階層で保持する State を [Context API](https://ja.reactjs.org/docs/context.html) を使って下層で参照する方法です。深いバケツリレーの回避のために使われることがあります。

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

Context API は確かに便利な機能ですが、バケツリレーの回避だけを解消するために使う機能ではありません。頻用はせず慎重に利用する機能です。React 公式ドキュメントの Context の説明から注意点を引用します。

> ## コンテクストを使用する前に
>
> コンテクストは主に、何らかのデータが、ネストレベルの異なる**多く**のコンポーネントからアクセスできる必要がある時に使用されます。コンテクストはコンポーネントの再利用をより難しくする為、慎重に利用してください。
>
> **もし多くの階層を経由していくつかの props を渡すことを避けたいだけであれば、[コンポーネントコンポジション](https://ja.reactjs.org/docs/composition-vs-inheritance.html)は多くの場合、コンテクストよりシンプルな解決策です。**

https://ja.reactjs.org/docs/context.html#before-you-use-context

他の観点としては、Server State ごとに Context を作成する必要があり、その管理が大変です。不意に Context を上書き（隠蔽）してしまう可能性もあります。「グローバル」変数がたくさんあったりローカル変数と名前が被る状況をご想像ください。

パフォーマンス観点では、不要な再レンダリングが起きてしまう大きな問題があります。Context API を使う際は注意するか [Constate](https://github.com/diegohaz/constate) のようなライブラリを使うなどして、常に再レンダリングを抑えるよう意識しなければなりません[^context-re-rendering]。

[^context-re-rendering]: 冒頭に記載したとおり筆者（の業務）の SPA のパフォーマンスへの要求は読者よりもおそらく低いですが、それでも生の Context が理由で重くなって対策することはありました。

### State 管理ライブラリ

Redux や Recoil などの Global State を扱おうとする State 管理ライブラリに格納しておいて、それを利用する方法です。データ取得ライブラリ登場以前はメジャーな方法でした。

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

Server State を使うだけならこれで良さそうです。一方、読み込み状態や成功と失敗の情報をそれぞれ定義するなどのボイラープレート[^suspense-error-boundary]や有効期限関連の処理は自分で実装しなければなりません。

[^suspense-error-boundary]: Suspense や ErrorBoundary を積極利用する場合は大丈夫かもしれません。

# データ取得ライブラリの利点

データ取得ライブラリを使う利点を紹介し、従来の課題がどのように解決または緩和されるか説明します。

これによって良い設計につながることも示します。全般的にボイラープレートも減ります。

## Server State 管理の難しさに対処しやすい

Server State 管理の技術的詳細[^detail]をデータ取得ライブラリに押し込められるため、Server State 管理の難しさに対処しやすくなります。余計なこと[^key]や不必要に細かいことを意識する必要はありません[^leaky-abstraction]。*戦略的にさほど重要でないこと*はライブラリに任せましょう。

[^detail]: 書籍 Clear Architecture の第 Ⅳ 部「詳細（detail）」の主張と近いです。
[^key]: キーという概念が新しく追加されているので要注意。
[^leaky-abstraction]: [漏れのある抽象化の法則（Leaky abstraction）](https://zenn.dev/nanagi/articles/0e899711611630#%E6%BC%8F%E3%82%8C%E3%81%AE%E3%81%82%E3%82%8B%E6%8A%BD%E8%B1%A1%E5%8C%96%E3%81%AE%E6%B3%95%E5%89%87%EF%BC%88leaky-abstraction%EF%BC%89)はあるため、完全には無視できないことにはご注意ください。

### 宣言的に書ける

データ取得ライブラリを使うと、データ取得処理を簡単に**宣言的**（Declarative）に書けます。React 等の利点としてよく出てくる「宣言的 UI」の「宣言的」と同じ意味です。

従来の命令的なデータ取得処理では、「マウントもしくはイベント発生のタイミングで、取得処理を実行し、State を更新する」と書くことで要求を実現します。これでも良いようにも見えますが、「こういうデータをいい感じに欲しいだけ」という要求に対しては余計なことを考慮しなければならないと考えることもできます。

```js :before.js
const [myState, setMyState] = useState();
useEffect(() => {
  async function fn() {
    const state = await fetchState(); // 2. 取得処理を実行し
    setMyState(state); // 3. State を更新する
  }

  fn();
}, []); // 1. マウントのタイミングで
```

データ取得ライブラリで宣言的に書くことによって、依存するデータのことだけ考えればよくなります。結果的に `useEffect` を使う必要が無くなります。ボイラープレートも減ります。

```js :after.js
const { data: myState } = useQuery(["state"], () => fetchState());
```

### Server State 管理を制御しやすい

データ取得ライブラリに備わる豊富な設定を利用することで、Server State 管理を制御しやすくなります。

```js
// TanStack Query
const {
  data, // Server State 本体
  status, // Server State の状態
  fetchStatus, // データ取得の状態
  error, // エラー内容
} = useQuery(["state"], fetchA, {
  refetchInterval: 10000, // 定期的なデータ再取得間隔
  refetchOnWindowFocus: true, // 再取得タイミングの追加
  staleTime: 0, // キャッシュで表示する裏でデータを取得するようになる時間（Stale-While-Revalidate）
  cacheTime: 1000, // キャッシュを破棄してデータを取得するようになる時間
  retry: 3, // 取得を 3 回リトライする
});
```

https://tanstack.com/query/v4/docs/reference/useQuery

データ取得ライブラリは定期的な取得やバックグラウンド処理、有効期限の扱い、リトライ処理をサポートし、かつ、柔軟に設定できます。各種状態も細かい観点で取得でき、そのままスムーズに UI とつなげられます。こうした様々な処理や概念が必要な場合でもすべてデータ取得ライブラリがやってくれるため、ボイラープレートは大きく削減されます。これらの概念そのものへの考慮を無くせるわけではありませんが、用意されたものをただ使うだけで良くなり楽になります。

## ユーザビリティを向上させやすい

データ取得ライブラリを使うと、ユーザビリティを向上させやすくなります。 ユーザビリティを向上させるために様々なテクニックの一部を簡単に実践できます。例えば、パフォーマンスを良くしたり読み込み表示を工夫したりするものです。

### パフォーマンスが良くなる

データ取得ライブラリは重複したデータ取得処理や不要な再レンダリングを防ぐため、（実際の）パフォーマンスが良くなります。こうした問題に自分で対処せずに済みます。

重複リクエストの排除や結果のキャッシュにより、ネットワークとメモリの無駄な消費を防ぎます。意識せずにデータ取得処理を書くと、複数箇所で実行されると重複したリクエストが複数回発生してしまいます。ネットワーク回線が貧弱な場合は他のリクエストを圧迫して読み込み速度を低下させてしまいます。

Server State 等の変化にのみ反応させることにより、メモリや処理能力の無駄な消費を防ぎます。State 管理の方法によっては、無駄なレンダリングが発生してしまいます。アプリケーションのスムーズな動作に必要な時間内で処理が終わらず、アプリケーションを遅くさせてしまいます。

### 読み込みの表示を工夫しやすい

[Suspense](https://ja.reactjs.org/docs/react-api.html#reactsuspense) の利用を含めて読み込みの設定や状態が豊富なため表示を工夫しやすいです。ユーザーにアプリケーションの動きが速いと感じさせることができ、（体感的な）パフォーマンスが良くなります。

プリフェッチの仕組みを使えば Render-as-You-Fetch[^render-as-you-fetch] を実現しやすく、従来の Fetch-on-Render での階段上の読み込みと表示よりも（実際に）速くなります。ページネーションでも力を発揮します。

[^render-as-you-fetch]: [Render-as-you-fetch パターンの実装｜ React の Suspense 対応非同期処理を手書きするハンズオン](https://zenn.dev/uhyo/books/react-concurrent-handson/viewer/render-as-you-fetch) が勉強になります。

ライブラリの支援があれば「楽観的な更新」の実現も簡単です。「楽観的な更新」によって、ユーザーの操作へのフィードバックを速くしたり読み込み表示による行動の中断を防いだりすることで、ユーザー体験を向上させます。

```js
// SWR
mutate("/api/state", update(newState), {
  optimisticData: newState,
  rollbackonError: true,
});
```

https://swr.vercel.app/ja/docs/mutation#%E6%A5%BD%E8%A6%B3%E7%9A%84%E3%81%AA%E6%9B%B4%E6%96%B0

## コンポーネント設計が良くなる

これらの利点とその理由を活用するとコンポーネント設計が良くなります。メンテナンス性が向上するため機能の新規開発や改善を高速に行えるようになり、ビジネス的にも採用的にも優位性が増します。

この設計的な利点はライブラリの説明や有志による解説にはあまり取り上げられないように見受けられます[^relay-keeps-iteration-quick]。筆者としては、SPA 設計に影響を及ぼす重要な観点だと思っていますので、その説明の節を設けることで利点を強調します。

[^relay-keeps-iteration-quick]: [Relay](https://relay.dev/) の "Keeps iteration quick" で触れられている程度です。

### コンポーネント間がより疎結合になる

データ取得ライブラリによってコンポーネント間の State の受け渡しや協調処理が減り、コンポーネント間がより疎結合になります。

疎結合になっているということは、他のコンポーネントのことを考慮せずともリファクタリングや機能実装タスクを実施しやすいということです。コンポーネントから Server State を使うときも、それらが他のコンポーネントから使われていることは意識せずに済みます。Server State 管理のミスが理由で他のコンポーネントを壊す心配もありません。コンポーネントの数が増えても Server State 管理は複雑にならず、開発がスケールします。

#### 分割統治

疎結合になることで分割統治を行いやすくなります。ここでは Atomic Design をコンポーネント単位や責務の分類として使ったときの Pages と Organisms の内容と関係を取り上げ、どのように分割統治できるか説明します。

Pages に集約させる理由が Server State に起因するの場合、データ取得ライブラリを使うと Organisms に Server State を使う役割を委譲可能となります。Pages に Server State を集約させるときの問題は、上述の「Server State の利用の観点」の内容のとおりです。Pages 内に並ぶ Server State 用コードは最小限に抑えられ、ボイラープレートも減ります。もともと Organisms で Server State を扱っていた場合でも、データ取得ライブラリを使うと同じ Server State を参照する外部のコンポーネントとの協調処理が不要になります。

:::details Before / After

（Pages において Server State を State 管理ライブラリから利用する場合の例）

```jsx :before.jsx
const Page = () => {
  const { data: article } = useArticle();
  const { data: relatedArticles } = useRelatedArticles();

  // other Server States...

  return (
    <div>
      <TitleOrganism title={article.title} />
      <ContentOrganism
        paragraph={article.paragraph}
        relatedArticles={relatedArticles}
      />
    </div>
  );
};

const TitleOrganism = ({ title }) => {
  return <header>{title}</header>;
};

const ContentOrganism = ({ paragraph, relatedArticles }) => {
  return (
    <main>
      <p>{paragraph}</p>
      <ul>
        {relatedArticles.map((article) => (
          <li>{article}</li>
        ))}
      </ul>
    </main>
  );
};
```

（Organisms において Server State をデータ取得ライブラリから利用する場合の例）

```jsx :after.jsx
const Page = () => {
  return (
    <div>
      <TitleOrganism />
      <ContentOrganism />
    </div>
  );
};

const TitleOrganism = () => {
  const { data: title } = useQuery(["article"], () => fetchArticle(), {
    select: (data) => data.title,
  });

  return <header>{title}</header>;
};

const ContentOrganism = () => {
  const { data: paragraph } = useQuery(["article"], () => fetchArticle(), {
    select: (data) => data.paragraph,
  });
  const { data: relatedArticles } = useQuery(["related-articles"], () =>
    fetchRelatedArticles(),
  );

  return (
    <main>
      <p>{paragraph}</p>
      <ul>
        {relatedArticles.map((article) => (
          <li>{article}</li>
        ))}
      </ul>
    </main>
  );
};
```

:::

Pages は子コンポーネントのための Server State 管理から解放され、Pages 特有の責務に集中できます。Organisms は自分が使う Server State の利用を（データ取得ライブラリを介して）自分で賄うことができます。データ取得ライブラリのキャッシュや反応性によってある程度は富豪的に書けるため、Organisms は外部のことを気にせず実装できます。

#### Suspense との関係

:::message
ここは明らかに独自研究的な主張ですので、苦手な方は飛ばしてください。
:::

唐突ですが Suspense と分割統治との関係を述べます。

Suspense と分割統治は非常に相性が良いと考えています。従来は、コード的には分割統治したい場合でも（読み込み時の）ユーザー体験を守るために分割できず、コードでも分割をあきらめるか親まで読み込み状態を伝えるか（ユーザー体験を妥協するか）が必要でした。データ取得ライブラリによるコード的な分割統治の容易さと Suspense による読み込み時のユーザー体験設定の柔軟さの組み合わせでもって、分割統治を推し進められます。

データ取得処理やライブラリに限らず Suspense そのものに言えることだと思いますが、Suspense は「読み込み中」というコンポーネント状態とその境界を扱いやすいです。子コンポーネントの各所で Suspend していても自コンポーネントの適切な場所に `<Suspense />` コンポーネントをかませれば、散らばったちぐはぐな読み込み表示でなく一体感のある読み込み表示が実現可能です。従来は親から子の中の読み込みの State を考慮して親で表示を切り替えるのは大変でしたが、今は `<Suspense />` コンポーネントので宣言的に書けます。

:::details Before / After

（分割統治で読み込み表示が分離する例）

```jsx :before.jsx
const Page = () => {
  return (
    <div>
      <TitleOrganism />
      <ContentOrganism />
    </div>
  );
};

const TitleOrganism = () => {
  const { data: title, isLoading } = useQuery(
    ["article"],
    () => fetchArticle(),
    {
      select: (data) => data.title,
    },
  );

  return isLoading ? <div>Loading title...</div> : <header>{title}</header>;
};

const ContentOrganism = () => {
  const { data: paragraph, isLoading: isContentLoading } = useQuery(
    ["article"],
    () => fetchArticle(),
    {
      select: (data) => data.paragraph,
    },
  );
  const { data: relatedArticles, isLoading: isRelatedArticlesLoading } =
    useQuery(["related-articles"], () => fetchRelatedArticles());

  return isContentLoading || isRelatedArticlesLoading ? (
    <div>Loading content...</div>
  ) : (
    <main>
      <p>{paragraph}</p>
      <ul>
        {relatedArticles.map((article) => (
          <li>{article}</li>
        ))}
      </ul>
    </main>
  );
};
```

（分割統治で Suspense を活用する例）

```jsx :after.jsx
// experimental suspense option is `true`.

const Page = () => {
  return (
    <Suspense fallback={<div>Loading page...</div>}>
      <div>
        <TitleOrganism />
        <ContentOrganism />
      </div>
    </Suspense>
  );
};

const TitleOrganism = () => {
  const { data: title } = useQuery(["article"], () => fetchArticle(), {
    select: (data) => data.title,
  });

  return <header>{title}</header>;
};

const ContentOrganism = () => {
  const { data: paragraph } = useQuery(["article"], () => fetchArticle(), {
    select: (data) => data.paragraph,
  });
  const { data: relatedArticles } = useQuery(["related-articles"], () =>
    fetchRelatedArticles(),
  );

  return (
    <main>
      <p>{paragraph}</p>
      <ul>
        {relatedArticles.map((article) => (
          <li>{article}</li>
        ))}
      </ul>
    </main>
  );
};
```

:::

Suspense とデータ取得ライブラリによる分割統治は相乗効果があると筆者は考えています。少なくとも、反応性によるコンポーネントの Suspend の発生（データ取得ライブラリによる分割統治から Suspense につながる）と読み込み表示の統合（Suspense がデータ取得ライブラリによる分割統治を補う）というつながりがあります。中央での Server State 管理が Suspense と相性が悪いから Server State を分散させるという見方でなく、（設計の利点のための）分割統治実現のために Suspense を利用するという見方を筆者は勧めます。

### コンポーネント内の見通しが良くなる

設計が良いということで親や子となるコンポーネントもあまり意識せずに済み、自分が管理する State やボイラープレートも減るため、コンポーネント内の見通しが良くなります。

ある程度の粒度の大きさのコンポーネントではコンポーネントとその子コンポーネントの内で Server State の利用が完結します。Server State の利用場所と描画場所とで距離が近づくため、そのコンポーネントで何を実現しようとしているか理解しやすくなります。

不必要な State やバケツリレーの Props などのボイラープレートも減ります。バケツリレーも必要なものだけに絞られるため、その回避のための仕組みは不要になります。

# 新しく生まれる問題や議論

データ取得ライブラリを導入することで発生する問題や議論（の中で筆者が見つけられたもの）を挙げます。新しい技術には小さくとも何かしらの欠点はありますので、実戦投入時にはトレードオフを考えどこを落とし所にするかという話になります。その検討の参考になればと思います。[^recoil]

[^recoil]: 公開時点では、Recoil のようなライブラリを基盤にしたデータ取得のプラグインか連携機能により、これらの問題が緩和されるのではないかと筆者は考えています。ただし、それをやったブログ記事等が検索にヒットしないため、この考えは的を外している可能性があります。

## キーという概念とキャッシュ

データ取得ライブラリを導入することでコードに混乱を招く可能性が高い概念は、おそらくはキーとその管理、および、キーとキャッシュの組み合わせではないかと思います。GraphQL クライアントでも間接的に現れます。

https://tkdodo.eu/blog/effective-react-query-keys

https://scrapbox.io/mrsekut-p/useQuery%E3%81%AEkey

副産物として、キーに対応するデータ取得処理のペアを管理することや、読み込みのトリガを取得処理の実行でなくキャッシュの無効化にすることも必要になります。

## エンドポイント間の依存管理

キャッシュに関連して、ある Mutation ではどのキャッシュを無効化すべきか、という考慮事項が生まれます。エンドポイント間の依存をどう管理してどうコードに確実に記載するかという課題です。

中規模以下の画面数や複雑度であれば工夫せずとも何とかやっていけるかもしれません。GraphQL クライアントではそもそもこの問題がないケースもあるでしょう。

## データ取得ライブラリの立ち位置

以下の記事の "React-Query — The pitfalls" は、データ取得ライブラリの立ち位置を考えるうえでかなり参考になります。

https://medium.com/duda/what-i-learned-from-react-query-and-why-i-will-not-use-it-in-my-next-project-a459f3e91887#ccd3

ここから文章を引用し、筆者の意見も添えます。（※訳ではありません）

### オーバーエンジニアリング？

> At present this isn’t the case with React-Query. **This lego piece is big, and can only fit in a specific way to a specific structure.**
> The concept of automating and managing the request flow could be relevant for various architectures. But the React-Query package was built to be consumed by a specific kind of architecture.

データ取得ライブラリは Server State 管理という限定的な問題を解決するライブラリとしては大きすぎるという指摘です。従来の State 管理ライブラリは、様々な問題を解決できたり様々なアーキテクチャで活用できたりしました。

そもそものところ、データ取得ライブラリが提供する高度な Server State 管理機能は、対象のアプリケーションでどのくらい必要でしょうか。小さいままのアプリケーションであれば、従来通りにデータを取得して従来通りに Server State 管理すれば十分かもしれません。

### Global State との分断

> React-Query suggests keeping server state and client state in different kinds of entities, accessed in a different way. But do they actually keep server state independent from client state?
> React-Query’s method for optimistic updates actually mixes server and client state. Treating server state as a reliable representation of server data is what counts, not having a different infrastructure to hold it.

既存のほとんどのデータ取得ライブラリは、Server State のみを扱うことを念頭に作られています。Client State やそれらとの連携のことはあまり考慮されていないため、Global State が Client State と Server State とに分断されてしまいます。これが問題となるのは、Client State が多く、かつ、それらが Server State と強く連携するアプリケーションでしょう[^global-state-heavy-app]。

[^global-state-heavy-app]: 筆者はこうしたアプリケーションの開発や設計に興味があるため、わざわざ取り上げています。

この記事では、「楽観的更新」の例で両 State が混在することについて指摘しています。他の具体例としては、TanStack Query にある Server State をどう Redux で管理する Client State に影響させるかと話があります。

この分断を避ける解決策の例としては [Recoil Relay](https://recoiljs.org/docs/recoil-relay/introduction) があります。

## 依存が増える

新しいライブラリを導入するとバンドルサイズは当然増えます。フロントエンドの宿命として、アプリケーションのバンドルサイズは常に懸念事項です。最近ではサプライチェーン攻撃も心配です。

筆者の SPA に対する関心と要求から生まれた個人的な意見として、増加が少しならバンドルサイズを増やしてでもメンテナンス性を上げ、開発スピードを維持または改善すべきだと考えています。また、闇雲に依存を避けるのではなく、無駄なくオーソドックスなライブラリを用いつつ（他のライブラリと同じく）バージョンのメンテナンスに時間を割くことが重要だと思います。戦略上、読み込み速度が重要でチューニングしている場合や Server State 管理に対して著名なデータ取得ライブラリを超えるカスタマイズ性が必要な場合は別だと思いますが。

## テスト

データ取得ライブラリによって、少なくともテストの基盤的なコードには影響がでます。データ取得ライブラリは自律的で高度な機能を備えており、かつ、それを使うコードがテスト対象になるためです。

https://tkdodo.eu/blog/testing-react-query

https://zenn.dev/akineko/articles/786fcefd759545

`QueryClient` のテスト向けインスタンスと API レスポンスのモックが必要です。

# まとめ

Server State とデータ取得ライブラリ、従来の Server State 管理方法を紹介しました。データ取得ライブラリの利点として、Server State 管理の難しさへの対処、ユーザビリティの向上、コンポーネント設計の品質向上を挙げました。最後に、データ取得ライブラリの導入時の注意点を挙げました。
