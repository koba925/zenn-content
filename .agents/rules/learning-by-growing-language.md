---
name: learning-by-growing-language
description: 「育てて学ぶ はじめての自作プログラミング言語」の原稿について
trigger: model_decision
---

Toil を題材とした書籍の執筆プロジェクトです。原稿は実装コードと常に整合している必要があります。

## 対象・関連プロジェクト

* Book 1（公開済み）: `books/learning-by-growing-language`（https://zenn.dev/kb84tkhr/books/learning-by-growing-language）
* Book 2（一部公開中）: `books/learning-by-growing-language-bk2`（https://zenn.dev/kb84tkhr/books/learning-by-growing-language-bk2）
* 処理系ソース: `/home/takahiro/projects/toil`（節ごと: `/home/takahiro/projects/toil/book`）
* サンプルコード: https://github.com/koba925/toil-book （節ごとのタグ例: `0101_constants`）

## 執筆の基本方針

* **ターゲット**: Python の基礎（リスト・辞書・クラス）は知っているが、言語処理系は初めての読者。
* **スモールステップ・楽しさ優先**: 厳密さよりわかりやすさと達成感を優先。機能追加の順序・構成では、1 度に複数概念（字句解析・構文解析・評価、参照と代入など）を導入せず認知負荷を極限まで分散させる。
* **トーン＆マナー**: 親しみやすい語り口とユーモアを保ち、各部・章・節末には次へのヒキを入れる。標準的な専門用語を用いつつ発展学習への注釈・コラムを適宜添える。

## セッション管理

コンテキスト肥大化を防ぐため、**節の区切り**または**作業スコープ変更**（原稿執筆 ⇄ Python 実装など）のタイミングでセッションを切り替える。終了時に `/lbgl-wrap-up-session`、開始時に `/lbgl-start-session` を使用する。また、本ファイル・`books/lbgl-context.md`・`lbgl-review/SKILL.md` は役割を重複させず、完了済みログを削りながら常にコンパクトに保つこと。
