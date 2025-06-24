# PR環境デプロイメントガイド

## 概要

このガイドでは、GitHub Pull Requestにラベルを追加することで、GCP Cloud Runに開発環境を自動デプロイする機能について説明します。

## 🚀 使用方法

### 1. 開発環境のデプロイ

1. Pull Requestを作成
2. PRに `deploy-dev` ラベルを追加
3. 数分後、PRコメントに開発環境のURLが投稿される

### 2. 環境の削除

#### 自動削除
以下のいずれかで自動削除されます：
- PRをクローズ（マージまたは却下）
- `deploy-dev` ラベルを削除

#### 手動削除
GitHub Actionsの「Cleanup PR Environment」ワークフローから手動実行も可能です：
1. [Actions](https://github.com/SotaYamaguchi/fact-checker/actions) → 「Cleanup PR Environment」
2. 「Run workflow」→ PR番号を入力して実行



## 🏗️ 動作の仕組み

- **サービス名**: `x-fact-checker-pr-{PR番号}`
- **環境変数**: `ENV=dev`, `PR_NUMBER={PR番号}`
- **リソース**: CPU 1コア、メモリ512Mi、最大2インスタンス
- **イメージタグ**: PRのHEAD commitハッシュ

## ⚙️ 設定

### 必要なGitHub Secrets

以下のSecretsが既に設定されています：
- `GCLOUD_SERVICE_KEY`: GCPサービスアカウントキー
- `PROJECT_ID`: GCPプロジェクトID
- その他アプリケーション固有のシークレット

### 必要な権限

GitHub Actionsワークフローには以下の権限が必要：
- `contents: read`: リポジトリ読み取り
- `id-token: write`: GCP認証用
- `pull-requests: write`: PRコメント投稿用

## 🛡️ セキュリティ考慮事項

### 1. 開発環境の分離

- 開発環境用の専用シークレットを使用することを推奨
- 本番データへのアクセスを制限
- 必要に応じてネットワーク分離を実装

### 2. リソース制限

- 開発環境のリソース使用量を制限（CPU、メモリ）
- コスト管理のため自動削除機能を活用

### 3. アクセス制御

現在は `--allow-unauthenticated` でデプロイしていますが、必要に応じて認証を追加できます：

```bash
# 認証が必要な場合
gcloud run deploy "$SERVICE_NAME" \
  --no-allow-unauthenticated \
  --add-iam-policy-binding \
  --member="user:developer@example.com" \
  --role="roles/run.invoker"
```

## 🎯 カスタマイズオプション

### 1. ラベル名の変更

`.github/workflows/deploy-pr.yml` の `deploy-dev` を別の名前に変更可能：

```yaml
if: contains(github.event.label.name, 'your-custom-label')
```

### 2. 環境別設定

開発環境専用の設定を追加：

```yaml
--set-env-vars="ENV=dev,PR_NUMBER=${{ github.event.number }},DEBUG=true"
```

### 3. 通知先の変更

SlackやTeamsに通知を送ることも可能：

```yaml
- name: Notify Slack
  uses: 8398a7/action-slack@v3
  with:
    status: success
    text: "PR #${{ github.event.number }} deployed to ${{ steps.get-url.outputs.service_url }}"
```

## 🗂️ ワークフロー構成

このシステムは2つのワークフローで構成されています：

### 1. `deploy-pr.yml` - デプロイメント
- **トリガー**: PRラベル追加（`deploy-dev`）
- **機能**: Cloud Runへのデプロイ、PRコメント投稿

### 2. `cleanup-pr-env.yml` - クリーンアップ
- **トリガー**: PRクローズ、ラベル削除、手動実行
- **機能**: 指定したPR環境の削除、詳細なログ出力

## 🚨 トラブルシューティング

### よくある問題

1. **デプロイが失敗する**
   - GCPサービスアカウントの権限を確認
   - Artifact Registryの設定を確認

2. **PRコメントが投稿されない**
   - `pull-requests: write` 権限を確認
   - GitHub Tokenの有効性を確認

3. **環境が削除されない**
   - 「Cleanup PR Environment」ワークフローのログを確認
   - 手動実行でPR番号が正しいか確認
   - GCPの権限を確認



### ログの確認

GitHub ActionsのWorkflowタブでデプロイ状況を確認できます：
`https://github.com/SotaYamaguchi/fact-checker/actions`

## 📈 改善案

### 1. 複数環境対応

```yaml
env:
  ENVIRONMENTS: '["dev", "staging"]'
```

### 2. データベース分離

PRごとに専用のデータベースを作成：

```yaml
- name: Create PR Database
  run: |
    gcloud sql databases create "pr-${{ github.event.number }}" \
      --instance="$INSTANCE_NAME"
```

### 3. E2Eテスト自動化

デプロイ後に自動テストを実行：

```yaml
- name: Run E2E Tests
  run: |
    npm test -- --baseUrl="${{ steps.get-url.outputs.service_url }}"
```

### 4. ブランチ保護ルール

本番ブランチへのマージ前に開発環境での検証を必須化：

```yaml
- name: Check PR Environment Status
  run: |
    # PRが正常にデプロイされていることを確認
```

## 💰 コスト管理

### リソース最適化

- **最小インスタンス**: 0（使用時のみ起動）
- **最大インスタンス**: 2（コスト制限）
- **CPU/メモリ**: 開発用に最小限

### クリーンアップ通知のカスタマイズ

Slackなど他の通知先も設定可能：

```yaml
- name: Notify Slack
  if: steps.check-conditions.outputs.should_cleanup == 'true'
  uses: 8398a7/action-slack@v3
  with:
    status: success
    text: "PR #${{ steps.set-pr.outputs.pr_number }} environment cleaned up"
```

## 📞 サポート

問題が発生した場合は、以下で確認してください：

1. [GitHub Actions ワークフロー](https://github.com/SotaYamaguchi/fact-checker/actions)
2. [GCP Cloud Run コンソール](https://console.cloud.google.com/run)
3. [GCP ログ](https://console.cloud.google.com/logs)

---

*このガイドは `cursor/explain-the-repository-details-1358` ブランチで作成されました。*