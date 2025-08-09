# Google Cloud セットアップガイド（初心者向け）

このガイドでは、Google Cloud Platform（GCP）でStreamlitアプリをCloud Runにデプロイするための初期設定手順を説明します。

## 前提条件

- Googleアカウント
- GitHubアカウント
- クレジットカード（Google Cloudの料金支払い用）

## 手順1: Google Cloudプロジェクトの作成

### なぜこの手順が必要なのか
Google Cloudでは、すべてのリソース（アプリケーション、データベース、ストレージなど）を「プロジェクト」という単位で管理します。プロジェクトは料金の請求単位でもあり、権限管理の境界でもあります。

### 何をするのか
あなた専用のGoogle Cloudプロジェクトを作成し、その識別子を取得します。

### 具体的な手順
1. [Google Cloud Console](https://console.cloud.google.com/) にアクセス
2. 右上の「プロジェクトを選択」をクリック
3. 「新しいプロジェクト」をクリック
4. プロジェクト名を入力（例：`my-streamlit-app`）
5. 「作成」をクリック

**取得する情報:**

- プロジェクトID（自動生成される文字列、例：`my-streamlit-app-123456`）
- プロジェクト番号（数字のみ、例：`123456789012`）

**環境変数を設定してコマンドを簡単にしましょう：**

```bash
# 以下の値を自分のプロジェクト情報に置き換えて実行
export PROJECT_ID="your-project-id-here"
export PROJECT_NUMBER="your-project-number-here"
export GITHUB_REPO="your-github-username/your-repo-name"

# プロジェクト番号を確認（PROJECT_IDを設定した後に実行）
gcloud projects describe $PROJECT_ID --format="value(projectNumber)"
```

**例：**
```bash
export PROJECT_ID="my-streamlit-app-123456"
export PROJECT_NUMBER="123456789012"
export GITHUB_REPO="marumarukun/streamlit-cloud-run-sample"
```

これらの情報は後で使用するのでメモしておいてください。

## 手順2: 必要なAPIの有効化

### なぜこの手順が必要なのか
Google Cloudでは、セキュリティのため必要な機能（API）だけを有効にする仕組みになっています。今回のデプロイでは、Cloud Run（アプリ実行）、Artifact Registry（Dockerイメージ保存）、Cloud Build（自動ビルド）、IAM（権限管理）の機能が必要です。

### 何をするのか
アプリのデプロイに必要な4つのAPIを有効にします。

### 具体的な手順
Google Cloud Consoleで以下のAPIを有効にします：

1. 左側メニューから「APIとサービス」→「ライブラリ」をクリック
2. 以下のAPIを検索して有効にする：

   - **Cloud Run Admin API** - アプリケーションを実行するサービス（「Cloud Run API」と表示される場合もあり）
   - **Artifact Registry API** - Docker画像を保存するレジストリ
   - **Cloud Build API** - 自動ビルドサービス
   - **IAM Service Account Credentials API** - 権限管理（「Service Account Credentials API」と表示される場合もあり）

各APIの有効化手順：
1. API名で検索
2. APIを選択
3. 「有効にする」をクリック

## 手順3: Artifact Registryリポジトリの作成

### なぜこの手順が必要なのか
StreamlitアプリはDockerコンテナとして動きます。そのDockerの「設計図」（イメージ）を保存する場所が必要です。Artifact Registryはその保管庫の役割をします。

### 何をするのか
Docker画像を保存するためのリポジトリ（保管庫）を作成します。

### 具体的な手順
1. Google Cloud Consoleで「Artifact Registry」を検索
2. 「リポジトリ」→「リポジトリを作成」をクリック
3. 以下を設定：
   - **名前**: `my-app-images`（任意の名前）
   - **形式**: Docker
   - **リージョン**: `asia-northeast1`（東京）または `asia-northeast2`（大阪）
4. 「作成」をクリック

**取得する情報:**

- リポジトリ名（例：`my-app-images`）
- リージョン（例：`asia-northeast1`）

## 手順4: Workload Identity の設定

### なぜこの手順が必要なのか
GitHub ActionsからGoogle Cloudにアクセスするには、安全な認証方法が必要です。従来はサービスアカウントキー（パスワードのようなもの）をGitHubに保存していましたが、Workload Identityを使うことで、より安全に認証できます。これは「GitHubからのアクセスだけを許可する専用の入り口」を作る仕組みです。

### 何をするのか
1. GitHub Actions専用のアクセス経路（Workload Identity Pool）を作成
2. GitHub Actionsがあなたのプロジェクトにアクセスできるよう権限を設定

### 4.1 Workload Identity Poolの作成

**ブラウザで設定する方法（推奨）:**
1. Google Cloud Consoleで「IAMと管理」→「Workload Identity連携」を検索
2. 「プールを作成」をクリック
3. プール名：`github-pool`
4. プールID：`github-pool`
5. 「続行」→「完了」

**コマンドラインで設定する方法:**
```bash
# Google Cloud CLIがインストールされている場合
gcloud iam workload-identity-pools create "github-pool" \
    --project="$PROJECT_ID" \
    --location="global" \
    --display-name="GitHub Actions Pool"
```

### 4.2 プロバイダーの作成

**何をするのか:** GitHubからのアクセスを識別するための設定を行います。

**コマンドラインで実行:**
```bash
gcloud iam workload-identity-pools providers create-oidc "github" \
    --project="$PROJECT_ID" \
    --location="global" \
    --workload-identity-pool="github-pool" \
    --display-name="GitHub Provider" \
    --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository" \
    --attribute-condition="assertion.repository == '$GITHUB_REPO'" \
    --issuer-uri="https://token.actions.githubusercontent.com"
```

### 4.3 サービスアカウントの作成と権限設定

**何をするのか:** GitHub Actionsが使用する「ロボットユーザー」を作成し、必要な権限を与えます。

```bash
# まず、gcloudでデフォルトプロジェクトを設定
gcloud config set project $PROJECT_ID

# サービスアカウント作成
gcloud iam service-accounts create github-actions \
    --display-name="GitHub Actions"

# Cloud Runへのデプロイ権限を付与
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:github-actions@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/run.admin"

# Artifact Registryへの書き込み権限を付与
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:github-actions@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/artifactregistry.writer"

# Service Account操作権限を付与
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:github-actions@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/iam.serviceAccountUser"

# Cloud Run開発者権限を付与
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:github-actions@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/run.developer"

# GitHubからサービスアカウントを使用する権限を付与
gcloud iam service-accounts add-iam-policy-binding \
    --role roles/iam.workloadIdentityUser \
    --member "principalSet://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/attribute.repository/$GITHUB_REPO" \
    github-actions@$PROJECT_ID.iam.gserviceaccount.com
```

## 手順5: GitHubリポジトリでの設定

### なぜこの手順が必要なのか
GitHub ActionsがあなたのGoogle Cloudプロジェクトを識別できるように、プロジェクトIDと番号をGitHubに安全に保存する必要があります。

### 何をするのか
プロジェクトの識別情報をGitHub Secretsに登録します。

### 5.1 GitHub Secretsの設定

GitHubリポジトリで以下のSecretsを設定：

1. リポジトリの「Settings」→「Secrets and variables」→「Actions」
2. 「New repository secret」で以下を追加：

**必要なSecrets:**

- `GCP_PROJECT_ID`: 環境変数で設定した `$PROJECT_ID` の値
- `GCP_PROJECT_NUMBER`: 環境変数で設定した `$PROJECT_NUMBER` の値

**確認コマンド:**
```bash
# 設定した値を確認
echo "Project ID: $PROJECT_ID"
echo "Project Number: $PROJECT_NUMBER"
```

### 5.2 ワークフローファイルの更新

**なぜこの手順が必要なのか:** 現在のワークフローファイルは他の人の設定になっているため、あなたの環境に合わせて変更する必要があります。

**何をするのか:** あなたのプロジェクト情報に合わせて設定ファイルを更新します。

`.github/workflows/deploy_run.yml` を以下のように更新：

```yaml
env:
  IMAGE_NAME: YOUR_REGION-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/YOUR_REPOSITORY_NAME/${{ inputs.app_name }}
  WORKLOAD_IDENTITY_PROVIDER: projects/${{ secrets.GCP_PROJECT_NUMBER }}/locations/global/workloadIdentityPools/pool/providers/github
  REGION: YOUR_REGION
```

**置き換える値:**

- `YOUR_REGION`: 手順3で選択したリージョン（例：`asia-northeast1`）
- `YOUR_REPOSITORY_NAME`: 手順3で作成したリポジトリ名（例：`my-app-images`）

## 手順6: デプロイテスト

### なぜこの手順が必要なのか
設定がすべて正しく行われているかを確認するため、実際にデプロイしてテストします。

### 何をするのか
GitHub Actionsを手動で実行して、アプリケーションがCloud Runに正常にデプロイされることを確認します。

### 具体的な手順
1. GitHubリポジトリの「Actions」タブ
2. 「Deploy Sample App to Cloud Run」ワークフローを選択
3. 「Run workflow」をクリック
4. App nameを入力（例：`my-streamlit-app`）
5. 「Run workflow」をクリック

## トラブルシューティング

### よくあるエラーと対処法

1. **権限エラー**: Workload Identityの設定を再確認
2. **APIが有効でない**: 手順2のAPI有効化を確認
3. **リポジトリが見つからない**: Artifact Registryの設定を確認

### 設定確認コマンド

```bash
# プロジェクト情報確認
gcloud projects describe YOUR_PROJECT_ID

# 有効なAPIの確認
gcloud services list --enabled

# Workload Identity Pool確認
gcloud iam workload-identity-pools describe github-pool --location=global
```

## 料金について

- **Cloud Run**: 使用分だけ課金（最小インスタンス0の場合、未使用時は無料）
- **Artifact Registry**: 保存容量に応じて課金
- **初回特典**: 300ドルの無料クレジットが利用可能

## 次のステップ

設定完了後、以下の情報を開発者に提供してください：

1. プロジェクトID
2. プロジェクト番号
3. 選択したリージョン
4. Artifact Registryリポジトリ名

これらの情報を使って、ワークフローファイルをあなたの環境用に更新します。
