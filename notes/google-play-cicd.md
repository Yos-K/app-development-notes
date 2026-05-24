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

## 全体構成

GitHub Actions から Google Play にアップロードする場合、構成要素は大きく 4 つに分かれる。

```mermaid
flowchart TB
    Repo[GitHub repository] --> Workflow[GitHub Actions workflow]
    Workflow --> Secrets[GitHub Environment Secrets]
    Workflow --> OIDC[GitHub OIDC token]
    OIDC --> WIF[Google Workload Identity Federation]
    WIF --> SA[Google service account]
    SA --> API[Android Publisher API]
    API --> Play[Google Play Console]
```

それぞれの責務は次のように分ける。

| 要素 | 責務 |
| --- | --- |
| GitHub Actions | テスト、ビルド、署名、AAB アップロードを実行する |
| GitHub Environment Secrets | 署名鍵と署名パスワードを安全に渡す |
| Workload Identity Federation | GitHub workflow に短期の Google 認証を与える |
| Play Console | 対象アプリへのアップロード権限と審査フローを管理する |

## 構築手順

手順は、先に「ビルドだけ通る状態」を作り、その後に「Play Console へアップロードできる状態」に広げると切り分けしやすい。

```mermaid
flowchart TD
    A[ローカルでビルドスクリプトを用意] --> B[GitHub Actions でテストとビルド]
    B --> C[署名済み AAB を artifact として保存]
    C --> D[Google Cloud の OIDC 認証を設定]
    D --> E[Play Console のサービスアカウント権限を設定]
    E --> F[Android Publisher API でアップロード]
```

### 1. ローカルでビルドをスクリプト化する

CI に載せる前に、ローカルで次の処理をコマンド化する。

- テストを実行する。
- debug APK を作る。
- release AAB を作る。
- release AAB に署名する。
- `versionName` と `versionCode` の整合性を確認する。

CI はローカル手順の置き換えではなく、ローカルで再現できる手順の自動実行にする。

### 2. GitHub Actions に CI workflow を置く

pull request と `main` push で、最低限次を実行する。

```mermaid
flowchart LR
    PR[Pull request] --> Test[Test]
    Test --> Debug[Debug build]
    Debug --> Artifact[Debug APK artifact]
```

CI の目的は、リリース以前に壊れた変更を止めること。
リリース署名鍵は不要なので、通常の CI には release 用 secret を渡さない。

### 3. GitHub Actions にリリース workflow を置く

リリース workflow は手動実行にする。

```mermaid
flowchart LR
    Manual[workflow_dispatch] --> Signed[Signed AAB]
    Signed --> Store[Artifact 保存]
    Signed -->|必要なときだけ| Upload[Play Console へアップロード]
```

手動入力として、次を持たせると運用しやすい。

| 入力 | 意味 |
| --- | --- |
| `channel` | free 版、pro preview などのビルド種別 |
| `track` | `alpha`、`beta`、`production` などの Play track |
| `status` | `draft`、`completed` などの release status |
| `upload_to_play` | Play Console へアップロードするか |

`upload_to_play=false` でも署名済み AAB を artifact として保存できるようにすると、CI/CD の問題と Play Console の問題を分けて確認できる。

### 4. GitHub Environment を分ける

GitHub Actions の environment は、用途ごとに分ける。

```mermaid
flowchart TD
    ReleaseBuild[release-build environment] --> BuildOnly[署名済み AAB 作成]
    PlayConsole[play-console environment] --> Upload[Play Console アップロード]
```

例:

| Environment | 用途 |
| --- | --- |
| `release-build` | 署名済み AAB を作るだけ |
| `play-console` | 署名済み AAB を作り、Play Console にアップロードする |

署名に必要な secret は両方に置く。
Play Console アップロードに必要な OIDC 設定は `play-console` にだけ置く。

### 5. 署名鍵を GitHub Secrets に登録する

公開リポジトリに署名鍵を置かない。署名鍵は base64 化して GitHub Environment Secret に登録する。

```mermaid
flowchart LR
    JKS[release keystore] --> Base64[base64 encode]
    Base64 --> Secret[GitHub Environment Secret]
    Secret --> Workflow[Release workflow]
```

登録する値の例:

| Secret | 意味 |
| --- | --- |
| `RELEASE_KEYSTORE_BASE64` | release keystore を base64 化した値 |
| `RELEASE_KEY_ALIAS` | 署名鍵の alias |
| `RELEASE_STORE_PASS` | keystore password |
| `RELEASE_KEY_PASS` | key password |

secret 名はプロジェクトごとに接頭辞を付けると衝突しにくい。

### 6. Google Cloud で Workload Identity Federation を設定する

サービスアカウントキーを発行せず、GitHub Actions の OIDC token を Google Cloud の短期認証に交換する。

```mermaid
sequenceDiagram
    participant GH as GitHub Actions
    participant Pool as Workload Identity Pool
    participant Provider as OIDC Provider
    participant SA as Service Account

    GH->>Provider: GitHub OIDC token
    Provider->>Pool: repository 条件を検証
    Pool->>SA: impersonation を許可
    SA-->>GH: 短期認証情報
```

重要な設定:

| 設定 | 例 |
| --- | --- |
| Issuer URL | `https://token.actions.githubusercontent.com` |
| Subject mapping | `google.subject=assertion.sub` |
| Repository mapping | `attribute.repository=assertion.repository` |
| 条件 | 特定リポジトリだけ許可する |

OIDC の許可条件は、少なくとも対象リポジトリに絞る。
可能なら branch や environment でも絞る。

### 7. Play Console でサービスアカウントを招待する

Google Cloud IAM だけでは Play Console を操作できない。
Play Console の `ユーザーと権限` で、サービスアカウントを対象アプリに招待する必要がある。

```mermaid
flowchart TD
    SA[Service account] --> IAM[Google Cloud IAM]
    SA --> PlayPerm[Play Console app permission]
    IAM --> Auth[API 認証]
    PlayPerm --> Edit[対象アプリの edit 作成]
```

必要な権限は運用方針によって変わるが、少なくとも対象アプリのテストトラックに AAB をアップロードし、リリースを編集できる権限が必要になる。

### 8. Android Publisher API を有効化する

Google Cloud プロジェクトで `Android Publisher API` を有効化する。

```mermaid
flowchart LR
    Project[Google Cloud project] --> API[Android Publisher API]
    API --> Enabled[Enabled]
```

API が無効な場合、GitHub Actions 側の認証が成功していてもアップロードは失敗する。

## workflow の構成

リリース workflow は、次の順に並べると失敗箇所を切り分けやすい。

```mermaid
flowchart TD
    Checkout[Checkout] --> Java[Set up Java]
    Java --> SDK[Set up Android SDK]
    SDK --> Tools[Install build tools]
    Tools --> Bundletool[Download bundletool]
    Bundletool --> Keystore[Decode keystore]
    Keystore --> Build[Build signed AAB]
    Build --> Artifact[Upload artifact]
    Build --> Auth[Authenticate to Google]
    Auth --> PlayUpload[Upload to Play]
```

アップロードしない実行でも artifact を残す。
これにより、署名済み AAB の生成と Play Console API の問題を分けられる。

## versionCode の扱い

Play Console に commit 済みの `versionCode` は再利用できない。

```mermaid
stateDiagram-v2
    [*] --> LocalBuild
    LocalBuild --> Uploaded: Play Console にアップロード
    Uploaded --> Rejected: 同じ versionCode で再アップロード
    Uploaded --> NextBuild: versionCode を増やす
```

CI/CD 化すると再試行が増えるため、次の運用を決めておく。

- Play Console にアップロードする前に `versionCode` を上げる。
- アップロード失敗が commit 前なら、同じ `versionCode` で再試行できる場合がある。
- AAB upload まで成功して edit commit で失敗した場合は、Play Console 側の扱いを確認する。
- 判断が難しい場合は、`versionCode` を上げて再試行する。

## リリース名の扱い

Google Play のリリース名には長さ制限がある。

```mermaid
flowchart LR
    SHA[full commit SHA] -->|長すぎる| Error[Play API error]
    Short[short SHA] --> OK[短い release name]
```

フル SHA を入れず、短い SHA や `versionName-versionCode` を使う。

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
