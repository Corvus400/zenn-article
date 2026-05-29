<!-- markdownlint-disable MD013 MD033 MD041 -->

![ヘッダー画像](./assets/readme/header.png)

# zenn-article

[Zenn](https://zenn.dev/) に投稿している技術記事を GitHub 連携で管理するリポジトリです。記事は `articles/` 配下に Markdown で配置し、`main` への push で Zenn へ自動デプロイされます。ファイル名がそのまま記事の slug となり、slug が一致する記事は新規投稿ではなく更新として扱われます。

[![Zenn](https://img.shields.io/badge/Zenn-3EA8FF?logo=zenn&logoColor=white)](https://zenn.dev/todayama_r)
[![Node.js](https://img.shields.io/badge/Node.js->=14-339933?logo=nodedotjs&logoColor=white)](https://nodejs.org/)
[![textlint](https://img.shields.io/badge/textlint-ja--technical--writing-3A0839)](./docs/writing-lint-policy.md)
[![License](https://img.shields.io/github/license/Corvus400/zenn-article)](./LICENSE)

ポートフォリオリポジトリのため外部 PR、一般的なサポート依頼、機能要望、通常のバグ報告は受け付けません。Issue は依存更新や公開リポジトリ運用上の衛生報告のために限定的に有効化しています。

公開ページ: <https://zenn.dev/todayama_r>

---

## 主な特徴

- **GitHub 連携での記事管理** — `articles/` の Markdown を `main` に push すると Zenn へ自動デプロイされる構成
- **再投稿にならない取り込み** — ファイル名 (= slug) を Zenn 上の既存記事と一致させることで、新規投稿ではなく更新として同期
- **Zenn CLI によるローカルプレビュー** — `npx zenn preview` で公開前の表示を確認
- **根拠ベースの文章 Lint** — textlint (preset-ja-technical-writing) + prh + markdownlint を、読みやすさ・有益さ・好感の定義に沿って取捨選択 ([docs/writing-lint-policy.md](./docs/writing-lint-policy.md))
- **Git hooks による品質ゲート** — husky の pre-commit (lint-staged) / pre-push で、変更・新規記事の文章とフォーマットを自動チェック
- **公開済み記事の保全** — 取り込み済みの既存記事は lint 対象外 (grandfather) とし、公開済みの内容を改変しない
- **公開前 hardening** — CC BY 4.0 License、Security Policy、Dependabot を適用

---

## 動かす

```bash
# 依存解決
npm install

# 公開前プレビュー (http://localhost:8000)
npm run preview

# 文章・フォーマットの Lint (新規・変更記事向け)
npm run lint        # textlint (preset-ja-technical-writing + prh)
npm run lint:md     # markdownlint
npm run lint:links  # リンク切れ検出 (no-dead-link)
npm run lint:fix    # textlint --fix
npm run format      # prettier --write
```

---

## 記事の管理

```bash
# 新しい記事を作成 (slug は自動採番)
npx zenn new:article

# 記事一覧 (slug / title / published)
npx zenn list:articles
```

- ファイル名 (拡張子を除く) が記事の **slug** になります。Zenn 上の既存記事と同じ slug にすると更新として同期され、新しい slug は新規投稿になります。
- frontmatter は `title` / `emoji` / `type` (`tech` または `idea`) / `topics` / `published` で構成します。
- Lint のルール選定方針とその根拠は [docs/writing-lint-policy.md](./docs/writing-lint-policy.md) を参照してください。

---

## License

記事コンテンツは [Creative Commons Attribution 4.0 International (CC BY 4.0)](./LICENSE) の下で公開しています。
