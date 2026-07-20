---
title: "remark-gherkin およびその Lint ルールを npm で公開した"
emoji: "🥒"
type: "tech"
topics: [ "remark", "gherkin", "markdown", "lint", "npm" ]
published: false
---

# TL;DR

remark-gherkin は、Markdown with Gherkin の構文をサポートする remark 用のプラグインです。

# 使い方

## npm package

Lint としては以下のように使える。

```sh
npm install remark-cli remark-gherkin remark-preset-lint-gherkin-lint

npm run remark .
```

- [ ] 実際にこれで動くか確認

## デモ

![img.png](/images/introduction-of-remark-gherkin/demo.png)

https://occar421.github.io/remark-gherkin/

# はじめに

- [ ] ここを書く

# 前提の技術

## Gherkin とは

- [ ] ここを書く

## Markdown with Gherkin とは

- [ ] ここを書く

## remark / remark-lint とは

- [ ] ここを書く

# やっていること

- [ ] ここを書く

- Markdown with Gherkin を mdast の拡張として parse 。
- このとき、mdast として有効な構造・データのままにする。
  - 他のツールを壊さない。
  - `data.gherkin` にてメタデータをつける。
  - remark-gherkin のツールは、このメタデータを主に利用する。

# 今後の展開

- [ ] ここを書く

- 他の Gherkin 向け lint ツールの移植
- SDD への活用
  - 別枠で EARS 記法のチェック
  - remark-gherkin をデータ元として、[Lean を使う個人的なモチベーション](https://zenn.dev/link/comments/5296eea0cac497)と接続する。
