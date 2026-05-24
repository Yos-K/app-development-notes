# Google Play CI/CD で学んだこと

## 原則

リリース経路は、開発者の端末や手作業に強く依存しない形にする。

```mermaid
flowchart TD
    Source[Git リポジトリ] --> CI[GitHub Actions]
    CI --> Signed[署名済み AAB]
    Signed --> Play[Google Play Console]
    Play --> Review[審査送信]
```

## 認証

サービスアカウントキーを保存するより、GitHub OIDC と Google Cloud Workload Identity Federation を優先する。

```mermaid
sequenceDiagram
    participant Workflow as GitHub workflow
    participant Identity as Workload Identity Provider
    participant Service as Service Account
    participant API as Android Publisher API

    Workflow->>Identity: GitHub token を交換
    Identity->>Service: 許可されたサービスアカウントを借用
    Service->>API: Publishing API を呼び出す
```

サービスアカウントには、2 種類の権限が必要になる。

- Google Cloud IAM 側の Workload Identity 権限。
- Play Console 側の対象アプリに対する操作権限。

## よくある失敗

```mermaid
flowchart TD
    A[Release workflow が失敗] --> B{エラー内容}
    B -->|API disabled| C[Android Publisher API を有効化]
    B -->|caller lacks permission| D[Play Console のユーザー権限を付与]
    B -->|version code used| E[versionCode を上げる]
    B -->|release name too long| F[リリース名を短くする]
```

## 運用ルール

Google Play のリリース名は短くする。

よい例:

- `LocalMD 1a2b3c4`
- `v0.1.0-14`

フルのコミット SHA を入れると、Play Console の長さ制限に引っかかる。

