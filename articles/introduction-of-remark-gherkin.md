---
title: "remark-gherkin およびその Lint ルールを npm で公開した"
emoji: "🥒"
type: "tech"
topics: ["remark", "gherkin", "markdown", "lint", "npm"]
published: false
---

# TL;DR

[`remark-gherkin`](https://github.com/occar421/remark-gherkin) は、Markdown with Gherkin（以下、MDG）を [remark](https://github.com/remarkjs/remark) で扱うためのプラグインです。Gherkin 記法（の Markdown 版）を、通常の Markdown と同じように検査できます。

あわせて、`gherkin-lint` 由来のルールを remark-lint として利用できるパッケージ群と、その preset である `remark-preset-lint-gherkin-lint` を npm に公開しました。

# はじめに

仕様を AI や開発ツールに渡しやすくし、仕様を中心にした Context Engineering / Harness Engineering につなげたいと考えています。そのためには、人がレビューしやすく、機械でも一貫して扱える仕様の書式が必要です。

Gherkin には、Markdown の記法で Feature を表現する [Markdown with Gherkin](https://github.com/cucumber/gherkin/blob/main/MARKDOWN_WITH_GHERKIN.md) があります。普段の Markdown と同じように差分を読み、リンクや補足を添えながら、シナリオの構造も保てる点が魅力です。

Markdown の解析・変換基盤には remark があります。そこで MDG を remark のエコシステムに接続し、既存の Markdown ツールを活かした lint を行えるようにしました。

# 使い方

## npm package

preset を使う場合は、以下のようにインストールします。 

※ この preset は [gherkin-lint](https://github.com/gherkin-lint/gherkin-lint) のルール群を移植しています。

```sh
npm install --save-dev remark-cli remark-preset-lint-gherkin-lint
```

MDG では、例えば次のように feature を書けます。

```md
# Feature: Eating cucumbers

## Scenario: Eat a cucumber

- Given there are 12 cucumbers
- When I eat 1 cucumber
- Then there are 11 cucumbers
```

次のコマンドで MDG を lint できます。

```sh
npx remark "**/*.features.md" --frail --use remark-preset-lint-gherkin-lint
```

アプリケーションやビルド処理に組み込みたい場合は、Unified API から preset を利用できます。

```js
import { remark } from "remark";
import remarkPresetLintGherkinLint from "remark-preset-lint-gherkin-lint";
import { reporter } from "vfile-reporter";

const file = await remark()
  .use(remarkPresetLintGherkinLint)
  .process(
    "# Feature: Eating cucumbers\n\n## Scenario: Eat\n\n- Given there are 12 cucumbers",
  );

console.error(reporter(file));
```

個別の Lint rule を設定・カスタマイズすることもできます。

## デモ

![Markdown with Gherkin の AST を表示するデモ](/images/introduction-of-remark-gherkin/demo.png)

[AST Explorer のデモ](https://occar421.github.io/remark-gherkin/)では、入力した MDG がどのような mdast になるかを確認できます。

# 前提の技術

Gherkin は、振る舞いを例で記述する BDD（Behavior-Driven Development）で使われます。通常の形式では、拡張子を .feature とするファイルの中で、以下のように書きます。

```gherkin
Feature: Staying alive
  This is about actually staying alive,
  not the Bee Gees song.

  Rule: If you don't eat you die
    ![xkcd](https://imgs.xkcd.com/comics/lunch_2x.png)

    @important @essential
    Scenario Outline: eating
      Given there are <start> cucumbers
      When I eat <eat> cucumbers
      Then I should have <left> cucumbers

      Examples:
        | start | eat | left |
        |    12 |   5 |    7 |
        |    20 |   5 |   15 |
```

MDG は、Gherkin を Markdown で書く dialect です。 .feature.md の拡張子で書きます。上記のコードは以下のように書けます。

```md
# Feature: Staying alive

This is about actually staying alive,
not the [Bee Gees song](https://www.youtube.com/watch?v=I_izvAbhExY).

## Rule: If you don't eat you die

![xkcd](https://imgs.xkcd.com/comics/lunch_2x.png)

`@important` `@essential`
### Scenario Outline: eating

* Given there are <start> cucumbers
* When I eat <eat> cucumbers
* Then I should have <left> cucumbers

#### Examples:

  | start | eat | left |
  | ----- | --- | ---- |
  |    12 |   5 |    7 |
  |    20 |   5 |   15 |
```

一般の Markdown の文書としても扱えるため、要件や仕様の背景、決定理由、関連資料へのリンクを、表現豊かに記述できます。 GitHub 等での扱いも Markdown ベースの方が有利でしょう。

## remark / remark-lint とは

[remark](https://github.com/remarkjs/remark) は Markdown の抽象構文木の一つの [mdast](https://github.com/syntax-tree/mdast) に変換し、pluggable に検査・変換・再出力できる基盤です（より厳密には [unified](https://unifiedjs.com/) の [unist](https://github.com/syntax-tree/unist) の Markdown サポート）。 rehype/hast を介して HTML を出力することもできます。

[remark-lint](https://github.com/remarkjs/remark-lint) は remark 上で動く Linter の基盤です。手元や CI 上で Markdown の構文をチェックできます。

これらは、ESTree/ESLint の Markdown 版と考えるとイメージしやすいと思います。

# remark-gherkin がやっていること

このリポジトリでは、それぞれ次の責務を持つパッケージを含めています。

- `mdast-util-gherkin`: mdast を解析して MDG の構文をメタデータに格納する
- `remark-gherkin`: MDG を認識する remark プラグイン
- `remark-lint-gherkin-*`: MDG のルールを検査する remark-lint プラグイン群
- `remark-preset-lint-gherkin-lint`: `gherkin-lint` 由来のルールをまとめた preset

MDG は mdast の拡張として解析されます。ただし、専用の構文木の要素の追加はせず、 plain な mdast として有効なデータを保ちます（メタデータを示す `data` に格納します）。これにより、既存の remark プラグインや Markdown の処理系がクラッシュしないよう努めています。

Lint ルールの多くは [`gherkin-lint`](https://github.com/gherkin-lint/gherkin-lint) と互換になるよう移植しました（remark の処理単位がファイル単位のみであることによる制約はある）。例えば以下のルールです。

- 重複した Feature 名や Scenario 名、タグの重複のチェック
- `Given`、`When`、`Then` の論理的な順序のチェック
- 未使用のパラメタのチェック
- 必要なタグの存在のチェック

# 今後の展開

- 他の Gherkin 向け lint ツールにある有用なルールを取り込む
- SDD（Specification-Driven Development）や GDD（Guarantee-Driven Development）の基盤にする
- mdast を起点に、ほかのツール（特に形式手法）と接続して文書のチェックや生成を行う
  - 元々これをやるために mdast で MDG を扱えるようにしたかった
  - [Lean を使う個人的なモチベーション](https://zenn.dev/link/comments/5296eea0cac497)にも書いてある
