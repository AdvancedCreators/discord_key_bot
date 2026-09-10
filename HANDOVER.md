# 引き継ぎ書 — Discord 鍵管理 Bot

> **対象読者**: このシステムを引き継ぐ運用担当者  
> **作成日**: 2026-06-20  
> **最終更新**: 2026-09-09

---

## 1. システム概要

| 項目 | 内容 |
|------|------|
| 名称 | Discord 鍵管理 Bot |
| 目的 | 部室の鍵の貸し借りを Discord チャンネル上で記録・管理する |
| 主なユーザー | 部員全員（ボタン操作） / 運用担当者（設定・再起動） |
| バージョン | schema_version 2（2026-06-20 時点） |

---

## 2. 稼働環境

### 2-1. Discord サーバー

| 項目 | 内容 |
|------|------|
| サーバー名 | ※ 実際のサーバー名を記入 |
| サーバー ID | ※ `.env` の `GUILD_ID` を参照 |
| 対象チャンネル | ※ `KEY_CHANNEL_ID` のチャンネル名を記入（例: `#鍵管理`） |
| チャンネル ID | ※ `.env` の `KEY_CHANNEL_ID` を参照 |

### 2-2. 実行サーバー

| 項目 | 内容 |
|------|------|
| ホスト名 / IP | 秘匿情報のため本書には記載しない。GitHub Secrets の `DEPLOY_HOST` と同じ値（運用担当者間で別途共有） |
| OS | ※ 未確認・記入 |
| 実行ユーザー（Bot本体） | `advan` |
| 実行ユーザー（自動デプロイ専用） | `deploy-bot`（ログインシェルは通常シェル、パスワードはロック。sudoは `systemctl restart key_bot` のみNOPASSWDで許可。`advan` と共有グループ `keybot` で `/opt/key_bot` を読み書き） |
| 稼働方式 | systemd サービス（ユニット名: `key_bot.service`） |
| 自動起動 | ※ 未確認・記入（`systemctl is-enabled key_bot` で確認可） |

### 2-3. ディレクトリ構成

```
/opt/key_bot/
├── key_bot.py                     # ボット本体
├── requirements.txt               # Python 依存パッケージ
├── README.md                      # セットアップ手順（開発者向け）
├── USAGE.md                       # 使い方ガイド（利用者向け）
├── HANDOVER.md                    # この引き継ぎ書
├── .github/workflows/
│   ├── ci.yml                     # 構文/致命的Lintチェック（push・PR時）
│   └── deploy.yml                 # main マージ時の自動デプロイ
├── .env                           # 機密設定（バックアップ必須・Git 除外）
├── venv/                          # Python 仮想環境
├── logs/
│   └── key_bot.log.*              # ローテーションログ（自動生成）
├── room_state.json                # 鍵の状態・履歴（自動生成・Git 除外）
└── nfc_tokens.json                # NFC 用トークン（自動生成・Git 除外）
```

---

## 3. 設定ファイル

### 3-1. `.env`（機密ファイル）

`.env` は Git 管理対象外です。**本番の値は運用担当者間で別途共有**してください。

```ini
# 必須
TOKEN=Discordボットトークン
KEY_CHANNEL_ID=操作対象チャンネルのID

# 強く推奨
GUILD_ID=サーバーID                     # スラッシュコマンドを即時反映させるため
ADMIN_USER_IDS=123456789,987654321      # /debug_* を使える管理者のユーザーID（カンマ区切り）

# オプション（デフォルト値で運用可）
NFC_PORT=8080                           # NFC HTTP サーバーのポート
LOG_MAX_BYTES=2097152                   # ログ 1 ファイルの最大サイズ (2MB)
LOG_BACKUP_COUNT=5                      # ログのローテーション世代数
```

> `.env` ファイルのバックアップ保管場所: ※ 保管場所を記入（例: 共有ドライブの○○フォルダ）

### 3-2. 主な設定値（コード内ハードコード）

変更する場合は `key_bot.py` の先頭付近を編集してください。

| 変数 | デフォルト | 意味 |
|------|-----------|------|
| `_DEFAULT_DAILY_HOUR` | `21` | 返却催促リマインドのデフォルト時刻（時） |
| `_DAILY_CLOSED_GRACE_MINUTES` | `30` | 返却催促を送る前の猶予（分） |
| `_DEFAULT_IDLE_HOURS` | `2` | 場所未報告リマインドのデフォルト時間（時間） |
| `_HISTORY_MAX` | `20` | 操作履歴の保持件数 |
| `_UNDO_TTL_SEC` | `60` | 取り消しボタンの有効秒数 |

---

## 4. 起動・停止・再起動

### systemd で管理している（本番はこちら）

```bash
# 状態確認
sudo systemctl status key_bot

# 再起動（設定変更後や落ちたとき）
sudo systemctl restart key_bot

# 停止
sudo systemctl stop key_bot

# ログをリアルタイムで見る
sudo journalctl -u key_bot -f
```

> `main` へのマージで自動的に `git pull` + 上記の再起動が実行されます（11章参照）。
> 手動での再起動は、緊急時や自動デプロイが失敗したときのみ。
> ゼロから手動起動する手順は [README.md](./README.md) の「セットアップ手順」を参照。

### 起動確認ポイント

ログに以下が出ていれば正常起動:

```
commands synced to guild <ID> (instant)
NFC HTTP server listening on port 8080
```

---

## 5. 依存関係とバージョン管理

| 項目 | バージョン |
|------|----------|
| Python | 3.10 以上（`str \| None` 構文を使用）。本番は 3.12 で稼働中 |
| discord.py | 2.7.1 |
| aiohttp | 3.13.5 |
| python-dotenv | 1.2.2 |
| tzlocal | 5.3.1 |

依存パッケージを更新するとき:

```bash
venv/bin/pip install -r requirements.txt   # インストール
venv/bin/pip freeze > requirements.txt     # バージョンを固定
```

---

## 6. 機能一覧

### 6-1. ボタン操作（Discord チャンネル内）

| ボタン | 動作 | 次に表示されるボタン |
|-------|------|------------------|
| 借りる | 自分を持ち主に設定 | 開ける / 返す / 受け取る / 持ち出す |
| 長期貸出 | 選択メニュー（明日/明後日/3 日後）→ 持ち主設定 + 長期貸出セット ※曜日を問わず常に表示 | 同上 |
| 開ける | state を `open` に | 閉める / 受け取る |
| 閉める | state を `closed` に | 開ける / 返す / 受け取る / 持ち出す |
| 返す | 持ち主をクリア、state を `closed` に | 借りる（初期画面） |
| 受け取る | 自分を持ち主に付け替え | 状況に応じたボタン |
| 持ち出す | 場所入力 Modal → state を `out` に | 開ける / 返す / 受け取る |
| 取り消す | 直前の操作を 1 分以内・本人限定で巻き戻す | — |

### 6-2. スラッシュコマンド / リマインド仕様 / NFC連携

[README.md](./README.md) の「スラッシュコマンド一覧」「リマインド仕様」「NFC連携」を参照。
内容はREADME.mdに一本化しており、本書には重複記載しない。

---

## 7. データ永続化

`room_state.json` のスキーマ・アトミック書き込み・排他制御などの詳細は
[README.md](./README.md) の「データ永続化」を参照。以下は運用上の注意点のみ:

- ボットを**再起動しても状態は保持**されます（ファイルが存在する限り）
- 壊れた場合は削除すると初期状態（施錠・持ち主なし）でリセット
- `nfc_tokens.json` は各ユーザーが `/nfc_register` を実行するたびに上書きされる（ユーザーID→トークンの対応表）

---

## 8. ログ

| 項目 | 内容 |
|------|------|
| 場所 | `logs/key_bot.log`（および `key_bot.log.1` ~ `key_bot.log.5`） |
| ローテーション | 2MB × 最大 6 ファイル（約 12MB 上限） |
| レベル | INFO（通常操作） / WARNING（軽微な失敗） / ERROR（起動失敗など） |

```bash
# リアルタイム表示
tail -f logs/key_bot.log

# 直近 50 行
tail -50 logs/key_bot.log
```

---

## 9. トラブルシューティング

| 症状 | 確認・対処 |
|------|-----------|
| Bot がオンラインにならない | `.env` の `TOKEN` が正しいか確認。ログに `SystemExit: 環境変数 TOKEN が設定されていません` が出ていれば `.env` の問題 |
| スラッシュコマンドが Discord に表示されない | `GUILD_ID` が設定されているか確認。未設定だとグローバル sync で最大 1 時間かかる。Discord を Ctrl/Cmd+R でリロード |
| `channel ... not found` エラー | `KEY_CHANNEL_ID` が正しいか、Bot がそのチャンネルの読み書き権限を持っているか確認 |
| ボタンが押せない（古いメッセージ） | 仕様。最新のメッセージまでスクロールすること |
| リマインドが来ない | `/reminder_status` で `daily_hour` / `idle_hours` が 0 になっていないか確認。ログに `reminder send failed` があれば Discord の権限問題 |
| NFC タッチが反応しない | `NFC_PORT` のファイアウォール開放を確認。ログに `NFC: unknown token` があればトークン再発行 |
| Bot が突然落ちた | `tail -50 logs/key_bot.log` でエラーを確認。systemd 運用なら自動再起動されているはず |
| `room_state.json` が壊れた | バックアップから復元。なければ削除して再起動（状態はリセットされる） |

---

## 10. 定期メンテナンス

| 作業 | 頻度 | 手順 |
|------|------|------|
| ログの確認 | 月 1 回程度 | `tail -50 logs/key_bot.log` でエラーがないか確認 |
| Bot トークンの確認 | 年 1 回 or 漏洩疑い時 | Discord Developer Portal で Reset Token → `.env` を更新して再起動 |
| 依存パッケージの更新 | 半年に 1 回程度 | `pip install -U -r requirements.txt` → 動作確認 → `pip freeze > requirements.txt` |
| `room_state.json` のバックアップ | 必要に応じて | `cp room_state.json room_state.json.bak` |

---

## 11. ソースコード管理

| 項目 | 内容 |
|------|------|
| リポジトリ | https://github.com/AdvancedCreators/discord_key_bot |
| ブランチ運用 | `main` ブランチが本番。機能ごとにブランチを切り、PRを`main`にマージする |
| Git 除外ファイル | `.env`, `logs/`, `room_state.json`, `nfc_tokens.json`, `venv/` |

### 11-1. 自動デプロイ（CI/CD）

仕組みの詳細は [README.md](./README.md) の「`main` マージでの自動デプロイ」を参照。
以下はこのインスタンス固有の値（README.mdには書かない本番限定情報）:

| 項目 | 内容 |
|------|------|
| デプロイ用アカウント | `deploy-bot`（ログインシェルは通常シェル、パスワードはロック。SSH鍵認証のみ） |
| sudo権限 | `deploy-bot` に `systemctl restart key_bot` のみ NOPASSWD 許可（`/etc/sudoers.d/`） |
| GitHub Secrets登録先 | `AdvancedCreators/discord_key_bot` の Settings → Secrets and variables → Actions |
| 実行状況の確認 | GitHub の Actions タブ（`Deploy` ワークフロー） |

**手動デプロイ**（緊急時・自動デプロイ失敗時）:

```bash
cd /opt/key_bot
git pull origin main
sudo systemctl restart key_bot
```

---

## 12. 担当者・連絡先

| 役割 | 氏名 | 連絡先 |
|------|------|--------|
| 現担当者（運用） | ※ 記入 | ※ 記入 |
| 前担当者 | ※ 記入 | ※ 記入 |
| 引き継ぎ先 | ※ 記入 | ※ 記入 |

---

## 13. 変更履歴

| 日付 | 変更内容 | 担当者 |
|------|---------|--------|
| 2026-06-20 | 初版作成 | ※ 記入 |
| 2026-09-05 | 長期貸出を金曜限定から曜日を問わず選べる方式に統一 | ※ 記入 |
| 2026-09-05 | `main`マージでの自動デプロイ(CI/CD)を追加、`AdvancedCreators` 組織へリポジトリ移管 | ※ 記入 |
| 2026-09-09 | 本章を実際の本番構成（パス・サービス名・デプロイ手順）に合わせて更新 | ※ 記入 |
