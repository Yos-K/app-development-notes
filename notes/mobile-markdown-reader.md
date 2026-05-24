# モバイル Markdown リーダーで学んだこと

## プロダクトの形

モバイルの Markdown リーダーは、アプリ内のファイル選択だけでなく、実際の受け渡し経路に対応して価値が出る。

```mermaid
flowchart LR
    Files[ローカルファイル] --> Reader[リーダー]
    Clipboard[クリップボード] --> Reader
    Share[他アプリから開く] --> Reader
    Reader --> Tabs[読みやすいタブ]
```

## UX

### 空の状態は次の操作を明確にする

空の画面は、次に何をすればよいかが分かる必要がある。

```mermaid
stateDiagram-v2
    [*] --> Welcome
    Welcome --> Reading: Markdown を開く
    Reading --> Welcome: すべてのタブを閉じる
```

ボタンに見えるものは、実際に押せるようにする。

### 文脈を失わせない

他アプリから 2 つ目のファイルを開いたとき、最初のファイルを黙って置き換えない。

```mermaid
flowchart TD
    A[外部アプリから開く] --> B{既存セッションがあるか}
    B -->|ある| C[新しいタブで開く]
    B -->|ない| D[最初のタブで開く]
```

### 一時的な内容は通常ファイルと分ける

クリップボードや選択テキストは、保存するまでは一時的な内容として扱う。

```mermaid
stateDiagram-v2
    [*] --> TemporaryDocument
    TemporaryDocument --> SavedDocument: 名前を付けて保存
    TemporaryDocument --> [*]: 保存せず閉じる
```

通常のファイルタブと同じ扱いにすると、保存導線やメッセージ表示が破綻しやすい。

## レンダリング

### テーブルの横スクロールは必要なときだけ見せる

横スクロールできる見た目は、実際に横幅が足りないときだけ表示する。

```mermaid
flowchart LR
    Table[テーブル幅] --> Compare[画面幅と比較]
    Compare -->|収まる| Static[スクロール表示にしない]
    Compare -->|はみ出す| Scroll[スクロール可能と分かる表示]
```

### 図はテーマに合わせて描画する

ダークテーマでは、Markdown 本文だけでなく図の線や文字色も調整する。

```mermaid
flowchart TD
    Theme[テーマ配色] --> Markdown[Markdown 本文]
    Theme --> Diagram[図のレンダラ]
    Markdown --> Page[表示ページ]
    Diagram --> Page
```

## ジェスチャー

ジェスチャーは文章で説明するより、小さなアニメーションで伝えるほうが分かりやすい。

```mermaid
flowchart LR
    Gesture[ジェスチャー候補] --> Preview[動きのプレビュー]
    Preview --> Action[割り当てる操作]
```

通常のスクロールがジェスチャーとして誤判定されないようにする。

