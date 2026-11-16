---
name: article-workflow
description: 日本語執筆から英語翻訳まで、言語・無料／有料・媒体に応じて記事制作スキルを選ぶ。複数媒体への展開や、制作経路を整理する依頼で使用する。投稿・販売操作は含まない。
---

# 日本語執筆から英語翻訳まで

基本は日本語で執筆し、確認した日本語稿から英語へ翻訳・再構成する。依頼された工程だけを行い、日本語記事の依頼に翻訳や有料版制作を自動追加しない。既存英語稿の改稿ではその稿を対象とする。

[共通編集方針](../../../docs/editorial-policy.md)、[制作手順](../../../docs/article-workflow.md)、[保存規約](../../../docs/content-layout.md)を参照する。英語化では[海外向け運用](../../../docs/overseas-content-workflow.md)も読む。

| 種類 | 日本語制作 | 英語展開 | 独立レビュー |
| --- | --- | --- | --- |
| 無料記事 | [japanese-article](../japanese-article/SKILL.md) | [english-article](../english-article/SKILL.md) | 日本語: [japanese-article-review](../japanese-article-review/SKILL.md)、英語: [english-article-review](../english-article-review/SKILL.md) |
| 有料記事 | [japanese-paid-article](../japanese-paid-article/SKILL.md) | [english-paid-article](../english-paid-article/SKILL.md) | 日本語: [japanese-paid-article-review](../japanese-paid-article-review/SKILL.md)、英語: [english-paid-article-review](../english-paid-article-review/SKILL.md) |
| 有料書籍・PDF化 | [japanese-paid-book](../japanese-paid-book/SKILL.md) | [english-paid-book](../english-paid-book/SKILL.md) | 日本語: [japanese-paid-book-review](../japanese-paid-book-review/SKILL.md)、英語: [english-paid-book-review](../english-paid-book-review/SKILL.md) |

無料・有料は作品の区分であり、全媒体で同じ販売方式を使う意味ではない。Qiitaは無料記事、Zennの有料制作は本として扱う。英語媒体の初出順は固定しない。会員向け記事・教材販売・無料配布などの提供方式と投稿仕様は、掲載準備時に指定条件と公式情報で確認する。

## 翻訳の責務

日本語側は日本語正本を制作し、英語側の `english-article`、`english-paid-article`、`english-paid-book` が[自然な英語への翻訳](../../../docs/english-translation.md)と読者向け調整を担当する。本スキルは種類に応じた引き渡し先を選ぶ。

## 著者の文体と翻訳元

既存の日本語執筆スキルと[過去の生原稿由来の文体プロファイル](../../../docs/author-voice-profile.md)を引き継ぐ。問い、試す理由、試行錯誤、発見、留保、自然な余談を保ち、体験や心情を創作しない。

翻訳元は `content.yaml` の `manuscripts.ja.file` が示す日本語公開稿。日本語稿がなければ対応する日本語執筆スキルで先に作る。著者確認専用の `ja_editorial` と日本語公開稿を混同しない。翻訳元の版、追加・省略・変更理由、未検証事項は日本語の `REVIEW.md` に記録する。英語では意味・条件・数値・コード・著者の視点を保持し、読者に必要な背景を補う。

同じ作品の日英稿は同じ作品内で管理する。異なる成果を持つ無料版と有料版は別作品として管理し、有料の非公開部分を無料へ混ぜない。同じ英語稿の媒体別コピーを正本として増やさない。

## 掲載準備

依頼された媒体向けの形式・紹介文・付属資料を整える。制作やレビューは投稿・販売開始・PRマージの許可にならない。公開は明示指示と公開条件に従い、媒体ごとの実公開を確認するまで公開済みに更新しない。自己レビューを独立レビューと記録しない。
