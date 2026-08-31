# 引き継ぎ書 — Discord 鍵管理 Bot

> **対象読者**: このシステムを引き継ぐ運用担当者  
> **作成日**: 2026-06-20  
> **最終更新**: 2026-06-20

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
| ホスト名 / IP | ※ 実際のホスト名または IP を記入 |
| OS | ※ 例: Ubuntu 24.04 LTS |
| 実行ユーザー | ※ 例: `botuser` |
| 稼働方式 | systemd サービス（またはその他 — 該当を記入） |
| 自動起動 | ※ はい / いいえ |

### 2-3. ディレクトリ構成

```
/path/to/discord_key_bot/          ← ※実際のパスを記入
├── key_bot.py                     # ボット本体
├── requirements.txt               # Python 依存パッケージ
├── README.md                      # セットアップ手順（開発者向け）
├── USAGE.md                       # 使い方ガイド（利用者向け）
├── HANDOVER.md                    # この引き継ぎ書
├── .env                           # 機密設定（バックアップ必須・Git 除外）
├── .venv/                         # Python 仮想環境
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

### systemd で管理している場合

```bash
# 状態確認
sudo systemctl status discord-key-bot

# 再起動（設定変更後や落ちたとき）
sudo systemctl restart discord-key-bot

# 停止
sudo systemctl stop discord-key-bot

# ログをリアルタイムで見る
sudo journalctl -u discord-key-bot -f
```

### 手動で起動している場合

```bash
cd /path/to/discord_key_bot        # ※実際のパスに変更
.venv/bin/python key_bot.py
```

バックグラウンド実行:
```bash
nohup .venv/bin/python key_bot.py > /dev/null 2>&1 &
```

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
| Python | 3.13 以上（`str \| None` 構文を使用） |
| discord.py | 2.7.1 |
| aiohttp | 3.13.5 |
| python-dotenv | 1.2.2 |
| tzlocal | 5.3.1 |

依存パッケージを更新するとき:

```bash
.venv/bin/pip install -r requirements.txt   # インストール
.venv/bin/pip freeze > requirements.txt     # バージョンを固定
```

---

## 6. 機能一覧

### 6-1. ボタン操作（Discord チャンネル内）

| ボタン | 動作 | 次に表示されるボタン |
|-------|------|------------------|
| 借りる | 自分を持ち主に設定 | 開ける / 返す / 受け取る / 持ち出す |
| 土曜まで借りる | 持ち主設定 + 長期貸出（土）※金曜のみ表示 | 同上 |
| 日曜まで借りる | 持ち主設定 + 長期貸出（日）※金曜のみ表示 | 同上 |
| 開ける | state を `open` に | 閉める / 受け取る |
| 閉める | state を `closed` に | 開ける / 返す / 受け取る / 持ち出す |
| 返す | 持ち主をクリア、state を `closed` に | 借りる（初期画面） |
| 受け取る | 自分を持ち主に付け替え | 状況に応じたボタン |
| 持ち出す | 場所入力 Modal → state を `out` に | 開ける / 返す / 受け取る |
| 取り消す | 直前の操作を 1 分以内・本人限定で巻き戻す | — |

### 6-2. スラッシュコマンド

| コマンド | 権限 | 説明 |
|---------|------|------|
| `/reminder_status` | 全員 | 現在の鍵の状態・リマインド設定を自分にだけ表示 |
| `/nfc_register` | 全員 | NFC タグ用の秘密トークンを発行 |
| `/reminder_daily hour:<時刻>` | 持ち主のみ | 返却催促リマインドの時刻を変更（0 で停止） |
| `/reminder_idle hours:<時間>` | 持ち主のみ | 場所未報告リマインドの時間を変更（0 で停止） |
| `/debug_friday on:<bool>` | 管理者のみ | 金曜モードの ON/OFF 切り替え（テスト用） |
| `/debug_reminder type:<daily/idle>` | 管理者のみ | リマインドを即時送信（テスト用） |

### 6-3. リマインド仕様

| 種類 | トリガー | 送信条件 | 抑制条件 |
|------|---------|---------|---------|
| 返却催促 (daily) | 毎日 設定時刻以降 | 未返却（`closed` / `out`）かつ最終操作から 30 分以上 + 当日未送信 | 長期貸出の最終日より前 |
| 場所未報告 (idle) | 毎分チェック | `closed` 状態（持ち主あり）で指定時間以上経過 | 長期貸出期間中 |

### 6-4. NFC 連携

- エンドポイント: `POST http://<サーバーIP>:8080/nfc`
- リクエストボディ: `{ "token": "</nfc_register で取得したトークン>" }`
- 動作: 鍵の状態を open ↔ closed でトグルし、Discord チャンネルに通知

---

## 7. データ永続化

### `room_state.json` — 鍵の現在状態

```jsonc
{
  "schema_version": 2,
  "state": "closed",        // "open" | "closed" | "out"
  "holder_id": null,        // 現在の持ち主の Discord ユーザーID
  "holder_name": null,
  "last_change_at": null,   // 最終操作の ISO8601 日時
  "last_message_id": null,  // 最新のボットメッセージID
  "out_location": null,     // 持ち出し中の場所
  "long_rent_until": null,  // 長期貸出の最終日 YYYY-MM-DD
  "debug_friday": false,
  "reminder": { ... },
  "history": []             // 末尾 20 件のみ保持
}
```

- ボットを**再起動しても状態は保持**されます（ファイルが存在する限り）
- 壊れた場合は削除すると初期状態（施錠・持ち主なし）でリセット

### `nfc_tokens.json` — NFC 認証トークン

- 各ユーザーが `/nfc_register` を実行するたびに上書きされる
- ユーザー ID → トークンの対応表

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
| リポジトリ | ※ GitHub リポジトリの URL を記入 |
| ブランチ運用 | `main` ブランチが本番 |
| Git 除外ファイル | `.env`, `logs/`, `room_state.json`, `nfc_tokens.json`, `.venv/` |

デプロイ手順（コードを更新するとき）:

```bash
git pull origin main
sudo systemctl restart discord-key-bot
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
