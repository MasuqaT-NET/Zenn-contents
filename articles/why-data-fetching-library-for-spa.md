---
title: "Data Fetching ライブラリを SPA に導入するとなぜ嬉しいのか"
emoji: "🧱"
type: "tech"
topics: ["frontend", "spa", "react"]
published: false
---

# TL;DR

# 文脈

## スコープ

CSR SPA no SSR ISR

React に限らないが筆者が React に慣れているので React で。

## State の種類

Server State をターゲットにする。

https://zenn.dev/yoshiko/articles/607ec0c9b0408d

https://zenn.dev/akfm/articles/react-state-scope

# Data Fetching ライブラリとは

データ取得ライブラリ

そのままサーバーからデータを取得するということだが、最近は Server State のキャッシュも含んでいる。

## ライブラリ

react-query SWR

GraphQL のクライアントの Apollo Client や Relay、 urql も同じ機能を持っている。

## HTTP クライアントとの違い

Axios や fetch

cache がある。React 等の UI ライブラリを念頭においていて、 Hook として提供されている。

# 利点

## 他の部分の無視

Atomic Design でいうところの Organism 単位。複雑さが減る。

cache が大体はいい感じにやってくれるので富豪的に書ける。

## Server State 管理の易化

ボイラープレートの削減。"画面" 遷移。非戦略的動作の外部調達。

## `useEffect` の打破

https://zenn.dev/dai_shi/articles/dee54f995e6e74

Suspense と相性もいいはず。

# 新しく生まれる課題

## エンドポイント間の関係性の管理

## 依存が増える
