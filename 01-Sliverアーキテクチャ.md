# Sliver C2 アーキテクチャ・仕組み解説

## 1. Sliver 全体アーキテクチャ概要

Sliverは Bishop Fox が開発した**オープンソースの Adversary Emulation / Red Team フレームワーク**です。  
Golangで書かれており、静的コンパイルされるため機能が非常に豊富ですが、その代償としてインプラントサイズが大きくなりがちです。

### 全体構成図（テキストベース）
```
┌─────────────────────────────────────────────────────────────────┐
│                        Operator Console                          │
│   (sliver クライアント / 複数オペレーター対応)                    │
└───────────────────────────────┬─────────────────────────────────┘
│ gRPC / Protobuf
▼
┌─────────────────────────────────────────────────────────────────┐
│                         Sliver Server                            │
│  - タスク管理 / 永続化DB / 複数オペレーター同時接続対応           │
│  - Implant管理 / Listener管理                                    │
└───────────────┬───────────────────────────┬──────────────────────┘
│                           │
┌───────▼────────┐          ┌───────▼────────┐
│  C2 Listener   │          │ Stage Listener │
│ (http/https/   │          │ (Stager専用)   │
│  dns/mtls/wg)  │          │                │
└───────▲────────┘          └───────▲────────┘
│                           │
│ 通常C2通信                 │ Stage 2配信
│ (Beacon/Session)          │ (shellcode)
│                           │
┌───────────────▼────────┐          ┌───────▼────────┐
│      Implant           │◄─────────┤    Stager      │
│  (Beacon or Session)   │          │ (小型初期       │
│  標的ホスト上で動作     │          │  ペイロード)   │
└────────────────────────┘          └────────────────┘
```

**主要レイヤー**:
- **Presentation Layer**: Operator Console
- **Application Layer**: Sliver Server（タスクキューイング、状態管理）
- **Transport Layer**: C2 Listener + Stage Listener
- **Agent Layer**: Implant（Beacon / Session）

## 2. 各コンポーネントの詳細定義と役割

### 2.1 Sliver Server

- **役割**: C2の心臓部。すべての状態を管理する。
- **主な機能**:
  - Listenerの起動・管理
  - Implantからの接続受付とタスクキューイング
  - 複数オペレーターの同時接続対応（gRPC）
  - 永続化（SQLiteなど）
  - Implantの生成リクエスト受付

### 2.2 Operator Console（sliverクライアント）

- 攻撃者が直接操作するCLI。
- `beacons`、`sessions`、`jobs`、`generate` などのコマンドを提供。
- ServerとgRPCで通信。

### 2.3 Implant（エージェント）

**定義**: 標的ホスト上で実際に実行される悪意のあるコード（ペイロード）。

- Golangで静的コンパイルされる。
- サイズ: 通常 **10〜16MB** 前後（機能による）。
- **2つの動作モード**を持つ（v1.5以降）:
  - **Beacon Mode**
  - **Session Mode**

### 2.4 Listener

**C2 Listener**:
- Implantが通常運用で接続してくるリスナー（http、https、mtls、dns、wg）。
- BeaconのチェックインやSessionの通信を受け付ける。

**Stage Listener**:
- Stagerが接続して**Stage 2（フルImplant）**をダウンロードするための専用リスナー。
- `stage-listener` コマンドで起動。
- 主にTCPまたはHTTPで動作。

### 2.5 Profile

**定義**: Implant生成時の「設計図」。

- C2 URL、間隔、jitter、OS/Arch、format、難読化設定などをまとめたもの。
- `profiles new` で作成し、再利用可能。
- Stager生成時にもProfileを指定して使う。

### 2.6 Stager

**定義**: 小さな初期ペイロード。フルサイズのImplantをメモリ上にダウンロードして実行する役割。

- サイズ: 数KB程度。
- Stager → Stage Listenerに接続 → Stage 2（フルImplant shellcode）をダウンロード → メモリ実行。

### 2.7 Transports

ImplantがC2と通信するためのプロトコル実装。
- HTTP/HTTPS
- mTLS
- DNS
- WireGuard
- Named Pipe / TCP Pivot（内部ピボット用）

## 3. Implantの動作モード比較

### Beacon Mode（非同期）

**仕組み**:
- `StartBeaconLoop` を使用。
- 設定された間隔（`--seconds`）＋ jitter で定期的にC2にチェックイン。
- チェックイン時に保留タスクを取得 → 実行 → 結果報告 → 再スリープ。

**プロセスレベルでの動き**:
1. 起動 → 初期化
2. メインゴルーチンが `time.Sleep(interval + jitter)` で寝る
3. 起床 → C2接続 → タスク確認
4. タスクがあれば別ゴルーチンで実行
5. 結果送信 → 接続切断 → 再スリープ

**特徴**:
- アイドル時のネットワーク・CPU使用が極めて低い
- 長期運用・ステルス運用に最適
- `interactive` コマンドで一時的に対話セッションを開ける

### Session Mode（同期・インタラクティブ）

**仕組み**:
- `StartConnectionLoop` を使用。
- 持続的接続またはLong Pollingでサーバーと常時近くにいる状態を維持。

**プロセスレベルでの動き**:
1. 起動 → 初期化
2. 即座にC2へ接続試行
3. 待機状態を維持（高頻度ポーリング or 持続接続）
4. サーバーからタスクが来たら即時実行・結果返信

**特徴**:
- コマンド実行レスポンスが非常に速い
- ネットワーク使用量が多い
- SOCKSプロキシや高度なピボット機能が使いやすい

### Beacon vs Session 比較表

| 項目               | Beacon Mode                  | Session Mode                  | 推奨用途             |
|--------------------|------------------------------|-------------------------------|----------------------|
| 通信スタイル       | 非同期（定期チェックイン）   | 同期・常時接続寄り            | -                    |
| アイドル時静かさ   | ◎                            | △                             | Beacon               |
| コマンドレスポンス | △（次のチェックイン待ち）    | ◎                             | Session              |
| 長期運用           | ◎                            | △                             | Beacon               |
| interactive操作    | `interactive`で一時的可能    | 最初から可能                  | 状況による           |
| モード変換         | Session化は可能              | Beacon化は不可                | -                    |

## 4. Stager と Staging の詳細な流れ

### Stagingの目的
- フルImplantが大きい（10MB超）ため、直接ドロップすると検知リスクが高い。
- 小さなStagerで初期侵入し、メモリ上でフルImplantを展開する。

### 実際の流れ（プロセス図）
```
1. Stager実行（標的）
↓
2. Stage Listenerに接続（TCP/HTTP）
↓
3. Stage 2（フルImplant shellcode）をダウンロード
↓
4. メモリ上に展開・実行
↓
5. Implant起動 → 通常のC2 Listenerに接続（Beacon or Sessionとして）
```

**Stage Listenerの役割**:
- ただ「Stage 2を渡すだけ」の短命リスナー。
- Profileを指定してどのImplantを渡すかを決める。

## 5. Implant生成プロセス（サーバー側）

1. `generate beacon` コマンド受付
2. Profile/フラグから設定を構築
3. Goクロスコンパイル実行（GOOS/GOARCH）
4. 設定情報（C2一覧、interval、jitter、モード情報）をldflagsなどでバイナリに埋め込み
5. 出力（exe / shellcode / dll など）

**重要なポイント**:
- モード（Beacon / Session）は**コンパイル時に固定**される。
- `--skip-symbols` を付けるとシンボル難読化がスキップされ、ビルドが高速化・サイズが小さくなる。

## 6. 通信の仕組み（深層）

### Beaconの通信サイクル
- 各チェックインで新しい接続を張る（短命）。
- メッセージ単位でセッション鍵による暗号化。
- HTTPの場合、URLをプロシージャル生成してランダム化。
- Jitter + Data Jitter でトラフィックパターンをばらつかせる。

### Sessionの通信
- Long Polling または持続的接続。
- サーバー側から比較的即時にタスクをプッシュ可能。

## 7. 主要用語集

| 用語            | 定義・説明                                                                 | 関連コンポーネント      |
|-----------------|----------------------------------------------------------------------------|-------------------------|
| **Implant**     | 標的で動くエージェント本体                                                 | 全般                    |
| **Beacon**      | 非同期チェックイン型Implant                                                | Implantの動作モード     |
| **Session**     | 同期・インタラクティブ型Implant                                            | Implantの動作モード     |
| **Stager**      | 小型初期ペイロード（Stage 1）                                              | 配信フェーズ            |
| **Stage 2**     | フルサイズのImplant                                                        | Staging                 |
| **Profile**     | Implant生成の設計図                                                        | 生成時                  |
| **Listener**    | C2通信を受け付けるサーバー側コンポーネント                                 | Server                  |
| **Stage Listener** | StagerがStage 2を取得するための専用リスナー                             | Staging                 |
| **Transport**   | ImplantとServer間の通信プロトコル実装                                      | Implant / Server        |
| **Jitter**      | チェックイン間隔のランダム揺らぎ                                           | Beacon                  |
| **interactive** | Beaconから一時的に対話セッションを開くコマンド                             | Beacon運用              |

## 8. まとめと設計思想

Sliverの設計思想は以下の通りです：

- **機能豊富さを優先** → インプラントサイズが大きくなる（だからStagerを強く推奨）
- **柔軟な通信モード** → BeaconとSessionを状況に応じて使い分け可能
- **運用しやすさ** → Profile、再利用、複数オペレーター対応
- **ステルス性** → Jitter、Data Jitter、URLランダム化、mTLS対応

**実運用での推奨**:
- 初期アクセス・長期潜伏 → **Beacon + mTLS or HTTPS + 適切なJitter**
- 積極的操作が必要な時 → `interactive` または **Session**
- サイズ制限がある配信 → **Stager + Shellcode形式**
