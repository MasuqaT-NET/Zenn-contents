---
title: "remark-gherkin およびその Lint ルールを npm で公開した"
emoji: "🥒"
type: "tech"
topics: ["remark", "gherkin", "markdown", "lint", "npm"]
published: true
---

# TL;DR

[`remark-gherkin`](https://github.com/occar421/remark-gherkin) は、Markdown with Gherkin（以下、MDG）を [remark](https://github.com/remarkjs/remark) で扱うためのプラグインです。Gherkin 記法の Markdown 版を、通常の Markdown と同じように検査できます。

あわせて、`gherkin-lint` 由来のルールを remark-lint として利用できるパッケージ群と、その preset である `remark-preset-lint-gherkin-lint` を npm に公開しました。

# はじめに

仕様を中心にした Context Engineering や Harness Engineering を実現したいと考えています。そのためには、人と AI の両方が理解できる形式の文書が必要ですが、その一つの候補が Gherkin 記法の .feature ファイルです。

Gherkin には、Markdown の記法で Feature を表現する [Markdown with Gherkin](https://github.com/cucumber/gherkin/blob/main/MARKDOWN_WITH_GHERKIN.md) という dialect があります。普段の Markdown と同じく、リンクや図を添えながら Scenario を書けるのが利点です。

Markdown の解析や変換の基盤には remark があります。remark で MDG をサポートするプラグインを実装し、独自の Lint および既存の Markdown ツールを活かした Lint を行えるようにしました。

# 使い方

## npm package

preset を使う場合は、以下のようにインストールします。 

```sh
npm install --save-dev remark-cli remark-preset-lint-gherkin-lint
```

MDG では、例えば次のように Feature を書けます。

```md
# Feature: Eating cucumbers

## Scenario: Eat a cucumber

- Given there are 12 cucumbers
```

次のコマンドで MDG を Lint できます。

```sh
npx remark "**/*.feature.md" --frail --use remark-preset-lint-gherkin-lint
```

アプリケーションやビルド処理に組み込みたい場合は、例えば以下のようにして preset を利用できます。

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

コマンドでも API でも、個別の Lint ルールを設定・カスタマイズすることもできます。

## デモ

![Markdown with Gherkin の AST を表示するデモ](/images/introduction-of-remark-gherkin/demo.png)

[AST Explorer のデモ](https://occar421.github.io/remark-gherkin/)では、入力した MDG がどのような [mdast](https://github.com/syntax-tree/mdast) になるか、また、どのような Lint の警告が出るかを確認できます。

# ベースとなる技術

## Gherkin

Gherkin は、振る舞いを例で記述する BDD（Behavior-Driven Development）でよく使われます。通常の形式では、拡張子を .feature とするファイルの中で、以下のように書きます。

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

### Markdown with Gherkin (MDG)

MDG は、Gherkin を Markdown で書く dialect です。ファイル名には .feature.md という拡張子を使います。上記のコードは以下のように書けます。

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

一般の Markdown の文書としても扱えるため、要件や仕様の背景、決定理由、関連資料へのリンクを、表現豊かに記述できます。 GitHub 等での扱い（特にプレビュー表示）も Markdown ベースの方が充実しています。

## remark / remark-lint

[remark](https://github.com/remarkjs/remark) は Markdown の抽象構文木 (AST) の一つの [mdast](https://github.com/syntax-tree/mdast) に変換し、pluggable に検査・変換・再出力できる基盤です（より厳密には [unified](https://unifiedjs.com/) の [unist](https://github.com/syntax-tree/unist) 向け Markdown サポート）。 rehype/hast を介して HTML を出力することもできます。

[remark-lint](https://github.com/remarkjs/remark-lint) は remark 上で動く Linter の基盤です。手元や CI 上で Markdown の構文をチェックできます。

これらは、ESTree/ESLint の Markdown 版と考えるとイメージしやすいと思います。

# remark-gherkin がやっていること

このリポジトリでは、それぞれ次の責務を持つパッケージを含めています。

- `mdast-util-gherkin`: mdast を解析して MDG の構文をメタデータに格納する
- `remark-gherkin`: MDG を認識する remark プラグイン
- `remark-lint-gherkin-*`: MDG のルールを検査する remark-lint プラグイン群
- `remark-preset-lint-gherkin-lint`: `gherkin-lint` 由来のルールをまとめた preset

MDG は mdast の拡張として解析されます。ただし、新しい要素の追加はせず、プレーンな mdast として有効なデータを保ちます（代わりにメタデータを示す `data` に格納します）。これにより、既存の remark プラグインや Markdown の処理系がクラッシュしないよう努めています。

今ある Lint ルールは [`gherkin-lint`](https://github.com/gherkin-lint/gherkin-lint) から移植しました（remark で実現できない機能はある）。例えば以下のルールです。

- 重複した Feature 名や Scenario 名、タグの重複のチェック
- `Given`、`When`、`Then` の順序のチェック
- 未使用のパラメタのチェック
- 必要なタグの存在のチェック

# 今後の展開

- 他の Gherkin 向け Lint ツールにある有用なルールを取り込む
- SDD（Specification-Driven Development）や GDD（Guarantee-Driven Development）の基盤にする
- mdast を起点に、ほかのツール（特に形式手法）と接続して文書のチェックや生成を行う
  - 元々はこれをやるために mdast で MDG を扱えるようにしたかった
  - [Lean を使う個人的なモチベーション](https://zenn.dev/link/comments/5296eea0cac497)にもメモを書いている
