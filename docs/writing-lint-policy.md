# 文章 Lint ポリシー

このリポジトリの textlint / markdownlint のルール採否は、機械的なデフォルトではなく
「人間が読みやすい・有益・好感を得やすい文章とは何か」を定義した上で、
その定義に資するルールを残し、過剰に厳格・有害なルールを除去する方針で決めている。

## 1. 良い文章の定義 (調査ベース)

| 観点 | 定義 | 出典 |
|------|------|------|
| 読みやすい | 一文が認知負荷の範囲に収まる (理想 40〜60 字、実用 80 字、上限 100 字)。読点・カンマ過多や長い漢字連続を避け、冗長表現・二重否定を排する。 | [一文の長さと読みやすさ (データのじかん)](https://data.wingarc.com/easy-reading-16542) |
| 有益 | 表記が正確で用語が統一され、曖昧さがない。冗長を削り情報密度を保つ。 | [技術文書向け textlint プリセット (efcl.info)](https://efcl.info/2016/07/13/textlint-rule-preset-ja-technical-writing/) |
| 好感を得やすい | 書き手の自己開示・感情・体験 (ストーリー) があり、語り口にリズムと温度がある。読者の悩みへの共感がある。 | [読まれる技術ブログの書き方 (模写修行)](https://moshashugyo.com/media/tech-blog-writing) / [共感される文章 (稼ぐ基盤)](https://kiban01.com/kyoukan-sareru-bun/) |

このリポジトリの記事は `type: tech` (翻訳・技術解説) と `type: idea` (イベント参加記などの体験談) が混在する。
`preset-ja-technical-writing` は**フォーマルな技術文書専用**に意図的に「少し厳しめ」へ振った設計であり、
公式自身がカスタマイズ前提と明言している ([公式 README](https://github.com/textlint-ja/textlint-rule-preset-ja-technical-writing))。
そのため体験談の語り口を損なうルールは「有害」と判断して除去する。

## 2. textlint ルールの採否

### 除去

| ルール | 除去理由 | 根拠 |
|--------|----------|------|
| `arabic-kanji-numbers` | 慣用句を機械変換で破壊する (実証: 「一つ一つ」→「1つ一つ」)。 | リポジトリ内で auto-fix 適用時に実害を確認 |
| `ja-no-weak-phrase` | 「思います」等を禁止するが、体験談では一人称の意見・自己開示が共感/好感の核。 | [模写修行](https://moshashugyo.com/media/tech-blog-writing) |
| `no-exclamation-question-mark` | 「！」「？」を禁止するが、ブログ的な語り口では engagement に寄与する。 | [稼ぐ基盤](https://kiban01.com/kyoukan-sareru-bun/) |
| `no-mix-dearu-desumasu` | ですます調の散文 + 箇条書きの体言止めという正当な文体に過剰発火する。 | 実測で 33 件の発火が箇条書きに集中 |

### 維持 (既定値)

`sentence-length` (100 字: 研究上の上限で過剰でない)、`max-ten` / `max-comma` (読点・カンマ過多回避)、
`max-kanji-continuous-len` (長い漢字連続回避)、`no-doubled-joshi` (助詞重複による読みにくさ回避)、
`ja-no-redundant-expression` (冗長表現の簡潔化)、`ja-no-successive-word` (単語重複=タイポ検出)、
二重否定の禁止、ら抜き言葉の禁止、逆接「が」の連続禁止。
これらは上記「読みやすい・有益」の定義に直接資するため残す。

## 3. markdownlint ルールの採否

### Zenn 独自記法と競合するため無効化

`MD013` (行長), `MD024` (兄弟見出しのみ禁止に緩和), `MD033` (インライン HTML), `MD034` (裸 URL), `MD041` (先頭見出し強制)。

### 根拠ベースで無効化

| ルール | 除去理由 | 根拠 |
|--------|----------|------|
| `MD045` (no-alt-text) | 隣接テキストで説明される装飾的画像は**空 alt が正しい** (alt を付けるとスクリーンリーダーに冗長)。本リポジトリの字幕付き連番スクリーンショットがこれに該当。 | [W3C WAI: Decorative Images](https://www.w3.org/WAI/tutorials/images/decorative/) |
| `MD025` (single-title) | Zenn は frontmatter に title を持つため本文の複数 H1 は問題にならない (`MD041` 無効化と同じ理由)。 | Zenn 仕様 |
| `MD060` (table-column-style) | テーブル列の整形スタイルは装飾的で表示に影響しない。 | — |

### 維持

`MD047` (末尾改行), `MD009` (行末空白), `MD022` (見出し前後空行), `MD012` (連続空行), `MD007` (リストインデント), `MD058` (テーブル前後空行) などの軽量な整形ルールは残す。

## 4. 既存記事の grandfather

取り込み済みの公開記事 10 件は Zenn で公開済みのため、**lint で一切改変しない (pristine 保全)**。
`.textlintignore` と `.markdownlint-cli2.jsonc` の `ignores` で除外し、ゲート (pre-push) の対象外とする。

- 利点: 上記の維持ルール (冗長表現チェック等) を**将来の記事には効かせつつ**、既存記事の発火でゲートを詰まらせない。`no-dead-link` も既存記事の大量の外部リンク確認から外れ、pre-push の不安定要因が減る。
- トレードオフ: grandfather した既存記事は**今後編集しても lint されない**。これらは完成済みの翻訳・過去イベントの記録であり編集頻度が低いため許容する。新規記事は別 slug のため通常どおり lint 対象。
