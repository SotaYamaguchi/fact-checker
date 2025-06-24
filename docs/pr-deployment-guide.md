# PR環境デプロイメントガイド

## 概要

Pull Requestにラベルを追加することで、GCP Cloud Runに開発環境を自動デプロイする機能です。

## 🚀 使用方法

### 1. 開発環境のデプロイ

1. Pull Requestを作成
2. PRに `deploy-dev` ラベルを追加
3. 数分後、PRコメントに開発環境のURLが投稿される

### 2. 環境の削除

#### 自動削除
- PRをクローズ（マージまたは却下）
- `deploy-dev` ラベルを削除

#### 手動削除
[GitHub Actions](https://github.com/SotaYamaguchi/fact-checker/actions) → 「Cleanup PR Environment」→ PR番号を入力して実行

## 🚨 トラブルシューティング

**デプロイが失敗する場合**
- [GitHub Actions](https://github.com/SotaYamaguchi/fact-checker/actions) でログを確認
- GCPサービスアカウントの権限を確認

**環境が削除されない場合**
- 「Cleanup PR Environment」ワークフローから手動実行
- [GCP Cloud Run コンソール](https://console.cloud.google.com/run) で直接確認

## 📝 備考

- サービス名: `x-fact-checker-pr-{PR番号}`
- リソース制限: CPU 1コア、メモリ512Mi
- 環境変数: `ENV=dev`

---

*問題が発生した場合は、GitHub Actionsのログまたは[GCP Cloud Run コンソール](https://console.cloud.google.com/run)を確認してください。*