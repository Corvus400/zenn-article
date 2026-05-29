# Security Policy

このリポジトリは Zenn に投稿している技術記事を GitHub 連携で管理するリポジトリです。

脆弱性の疑いがある内容は、このリポジトリの GitHub Security Advisories から非公開で報告してください。非公開で扱うべき詳細を public Issue、Pull Request、コメント、ログ、スクリーンショットに投稿しないでください。

報告には、不要な秘密情報、認証情報、アクセストークン、private URL、ローカルマシンの絶対パス、個人メールアドレス、非公開の個人情報を含めないでください。

## Scope

対象:

- `articles/` 配下の公開記事コンテンツ
- Zenn GitHub 連携 (自動デプロイ) の構成
- Lint / Git hooks (textlint・markdownlint・prettier・husky) と npm 依存
- GitHub Actions / dependency update configuration

対象外:

- 一般的なサポート依頼
- 記事内容の文言・表現の改善提案
- 外部サービス (Zenn・GitHub 自体) の脆弱性

## Repository Operation

このリポジトリはポートフォリオ用途です。外部 PR、一般的なサポート依頼、機能要望、通常のバグ報告は受け付けません。

Issue は、依存更新や公開リポジトリ運用上の衛生報告のために限定的に有効化しています。公開 Issue では、秘密情報、認証情報、アクセストークン、private URL、ローカルマシンの絶対パス、個人メールアドレス、非公開の個人情報、脆弱性の詳細を扱いません。

依存更新とセキュリティ通知のために Dependabot / GitHub Security 機能を利用します。
