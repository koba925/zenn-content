---
name: learning-by-growing-language
description: 「育てて学ぶ はじめての自作プログラミング言語」の原稿について
trigger: model_decision
---

Toil を題材として書籍「[育てて学ぶ はじめての自作プログラミング言語](https://zenn.dev/kb84tkhr/books/learning-by-growing-language)」を執筆中です。まだドラフトですが、第 1 部を公開し、第 2 部を執筆中です。

* `books/learning-by-growing-language`： 公開済みの原稿
* `books/learning-by-growing-language-draft`： 作成中・公開予定のドラフト原稿

## 関連プロジェクト

本書のソースコードは以下のプロジェクトにあります。本書の原稿と整合性が取れている必要があります。

* Toil 言語処理系のソースコード: `/home/takahiro/projects/toil`
  * 節ごとのソースコード: `/home/takahiro/projects/toil/books`
* サンプルコード https://github.com/koba925/toil-book
  * 節ごとのソースを参照できるようタグを打っています。
    例： https://github.com/koba925/toil-book/tree/0101_constants

## セッション管理

コンテキストの肥大化を防ぎ、高い精度で作業を行うため、以下のタイミングでチャットをクリアして新しいセッションに切り替えること。セッション終了時は `/lbgl-wrap-up-session`、開始時は `/lbgl-start-session` を活用する。

1. **タスクの区切り**: 1 つの節（例：4.1 節など）の執筆やレビューが完了し、次の節へ移る時。
2. **スコープの変更**:「原稿の執筆・推敲」から、Toil 言語本体の「Python コードの実装・デバッグ」などへ作業対象が大きく変わる時。
