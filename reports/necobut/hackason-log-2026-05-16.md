# カチャカハッカソン作業ログ 2026-05-16

ROS Japan UG #62「Vibe Codingしばりのハッカソン カチャカ開発編」@WHILL本社

## 概要

Telegram → 秘書AI「ネビィ」→ カチャカ操作 / StackChan発話・画面表示の統合システムを1日で構築。

## 達成した機能

| 機能 | 発火例 |
|---|---|
| 手動操作 | 「カチャカ前進」「止まれ」「右」 |
| ステータス | 「カチャカ調子は？」 |
| 自律走行 | 「ポイントAに行って」「ホームに戻って」 |
| 走行中の状況描写 | StackChan が VLM 経由で「机の前に椅子があるよ」等を発話 |
| 到着通知 | 「○○に着いたよー！今は△△が見えるよ」 |
| **画面共有** | カチャカ前カメラ画像を 3秒おきに StackChan画面に表示、到着10秒後にアバター復帰 |

---

## システム構成

```
[User Telegram]
     │ "カチャカ ポイントAに行って"
     ▼
[秘書 on 自宅Mac (mbpm1)]  ← Claude Code (Sonnet 4.6) + Telegram plugin
     │
     ├── reply「ポイントAに向かうねー」
     │     ↓ tts-hook.sh (PostToolUse)
     │     ↓ POST /speak
     │   [tts-server.py] ─ open_jtalk ─→ PCM16 24kHz queue
     │     ↓ Tailscale Funnel
     │   [StackChan] poll /audio → 発話
     │
     └── Bash: ~/bin/kachaka-goto.sh ポイントA
           │
           ↓ SSH (鍵認証)
        [会場Mac (MBPM4)]
           │
           ↓ docker exec
        [ros2_bridge コンテナ on OrbStack]
           │ Action: kachaka_command/execute (MOVE_TO_LOCATION)
           ↓ gRPC
        [kachaka 本体]

        ──── Python watcher が並行動作 ────
        ① subscribe /kachaka/front_camera/image_raw/compressed (3秒おき POST /screen)
        ② subscribe /kachaka/odometry/odometry (到着検知用)
        ③ 走り始め/20秒おき/到着時: 画像を LM Studio (Qwen3.6 35B MLX) に投げて
           日本語描写を取得 → /speak に POST → StackChan が喋る
        ④ 到着10秒後: POST /screen/clear → StackChan画面をアバターに戻す
```

---

## 関与するリポジトリ

| リポジトリ | 役割 | ブランチ |
|---|---|---|
| `kachaka-hackason` | 本リポ。kachaka-api clone + ros2_bridge 起動・実機接続情報 | main |
| `claude-telegram-test` | 秘書システム（Telegram + Claude Code + TTS） | `hackathon-kachaka` |
| `KSstackchan` (stackchan-custom) | StackChan ファームウェア (ESP-IDF v5.5) | `hackathon-kachaka` |

---

## 環境

- 会場マシン: macOS, Apple Silicon (arm64)
- コンテナ: **OrbStack** (Docker Desktop ではない)
- IDE/Agent: Claude Code (Sonnet 4.6)
- VLM: LM Studio + Qwen 3.6 35B MLX
- 通信: Tailscale (Funnel & 直接接続)
- Kachaka: API設定ON、ファームウェアブリッジは公式の Docker image

---

## タイムライン（要約）

### 1. 環境調査・準備
- Claude Code起動、`rosjp-vibe-coding-hackathon` リポを調査
- 4つのエージェントスキル把握（bridge-setup, env-setup, topics, control）
- macOS で Docker = **OrbStack** 構成と判明（Docker Desktopではない、auto-detectが効く）
- `kachaka-api` を clone、`kachaka-ros2-client` イメージを Dockerfile からビルド
- `start_bridge.sh -d` でブリッジ起動（kachaka本体のローカルIPに接続）

### 2. 基本接続確認
- `ros2 topic list` で約25トピック確認
- `/kachaka/robot_info/battery_state` でバッテリー93% 取得
- `grab_image.py` で `/kachaka/front_camera/image_raw/compressed` から JPEG 1枚取得・表示

### 3. 連携設計（Plan モードでアーキ確定）
- 3プロジェクト連携、ブランチ運用で巻き戻し可能な設計に
- 当初の「`--sync` でStackChan発話と動作を同期」は SSH遅延で15秒かかったため後に撤廃

### 4. クロスMac構成（案B）
**発見**: 秘書システムは自宅Mac (mbpm1) で常駐、会場Mac (MBPM4) には未デプロイ。Tailscale で両Macは常時接続。

**選択**: 案B — mbpm1 から SSH 越しに MBPM4 の docker を操作

実装したスクリプト:
- `bin/kachaka-move.sh` — 前進/後退/旋回/停止 (auto-detect で local/SSH 切替)
- `bin/kachaka-status.sh` — ブリッジ状態 + バッテリー% 報告
- `bin/kachaka-goto.sh` — 登録地点への自律走行（後に大幅拡張）

### 5. SSH鍵セットアップ
- mbpm1 → MBPM4 へ key auth 必須
- mbpm1 で ed25519 鍵生成 → 公開鍵を MBPM4 の `~/.ssh/authorized_keys` に追加
- 動作確認: `kachaka-status.sh` で「ブリッジ稼働中 (MBPM4経由)、バッテリー○%、絶好調」

### 6. 非対話SSH の罠
- `which docker` が SSH ログインで通らず「command not found」
- OrbStack の `/usr/local/bin/docker` を絶対パスで呼ぶよう修正
- `KACHAKA_DOCKER_BIN` 環境変数で上書き可能化

### 7. CLAUDE.md 配信フロー
- `setup-mac.sh` が `bin/*.sh` を `~/bin/` に、`CLAUDE.md.mac` を `~/CLAUDE.md` にコピーする方式
- git pull だけでは反映されない → `cp` で同期する手順を確立

### 8. 自律走行（kachaka_command Action）
- `goal_pose` の publish では完了検知できないことが判明
- `/kachaka/kachaka_command/execute` Action server を発見
- `MOVE_TO_LOCATION_COMMAND(7)` と `RETURN_HOME_COMMAND(8)` を使用
- 完了監視を bridge container 内 Python (`docker exec -d`) でバックグラウンド実行

### 9. 到着通知バグ修正
**バグ1**: NOTIFY_HOST のデフォルトが逆。container内watcher は常に tts-server (mbpm1) を指す必要があった
**バグ2**: container内 DNS では Tailscale MagicDNS が引けない → IP直叩きに変更

### 10. 60秒タイムアウト問題
- bridge側に `kWaitForPendingTimeout=60s` があり、長距離移動で action が早期 ABORTED
- 「到着できませんでした」と誤報告される
- 対策: `/kachaka/odometry/odometry` を購読、action result は参考程度、**kachaka が実際に2秒間静止するまで観測継続**、絶対上限 5分

### 11. VLM 統合
- LM Studio (qwen3.6-35b-a3b-ud-mlx) で画像描写を取得 → StackChan発話
- プロンプト: 「カメラに映ってるものを20文字前後のフレンドリーな喋り言葉にして」
- 走り始め「じゃ出発するよー！今ね、〇〇」
- 走行中（20秒おき）「途中だけど、〇〇」
- 到着「ポイントAに着いたよー！今は、〇〇」

### 12. StackChan画面に画像表示（最大の山場）

**サーバ側 (`tts-server.py`)**:
- `/screen` POST: JPEG受信 → Pillow で 320×240 letterbox リサイズ → スロット保持
- `/screen` GET: 保持中 JPEG を返す (204 if 空)
- `/screen/clear` POST: スロットクリア

**watcher側 (`kachaka-goto-watcher.py`)**:
- 別スレッドで 3秒おきに最新画像を `/screen` に POST
- 到着+10秒 で `/screen/clear` POST
- VLM 解説とは独立

**Tailscale Funnel**:
- `funnel-watchdog.sh` の PATHS に `/screen` を追加
- Funnel を replay して `/screen` を Tailscale で公開

**Firmware (`screen_client.cpp` 新規)**:
- 3秒おきに `/screen` をポーリング
- 200 with JPEG → `jpeg_dec::decode_to_lvgl()` → `lv_image_create(lv_screen_active())`
- 204 → `lv_obj_delete()` でアバターに戻る

**ビルドハマりどころ**:
1. CMake の `file(GLOB)` が新規 .cpp をキャッシュで拾わない → `idf.py reconfigure` 必要
2. `LvglLockGuard` not declared → `<hal/hal.h>` include 漏れ
3. `--no-feedback` フラグが ROS2 にない → 削除
4. `success: True` (Pythonの大文字Bool) vs `success: true` (YAMLの小文字) → grep の `-i` 修正

**ランタイムハマりどころ**:
5. **再起動ループ** — `audio_codec init` で `ESP_ERR_NO_MEM`。screen_client task の存在で internal RAM が圧迫
6. **task 起動しない** — stack 16KB が free SRAM 14KB を超えて malloc 失敗 → 8KB に削減
7. **audio_codec OOM** — tts_client の `Board::GetAudioCodec()` が遅延初期化。screen_client がメモリ食う前に init 必要
8. **解決策**: `screen_client_start()` を **60秒 delay** して起動（tts_client が音声を1度再生してから）

**最終的に成功**:
- Telegram で起動した画像表示用 TTS（「画面表示のテスト中だよー」）でaudio_codec初期化
- 60秒経過後 screen_client task が起動
- `/screen` ポーリング開始、サーバから JPEG 取得・表示成功

### 13. 動作確認
- StackChan は約3.5秒間隔で正しく GET /screen
- サーバが返す JPEG (16.6KB) を完全に再生
- 「画面に画像が出た！」確認

---

## 主要な技術判断

1. **既存運用サービスを極力触らない**
   - tts-server.py は触らずに済む設計から始めた（最終的には `/screen` だけ追加）
   - tts-hook.sh / launchd plist は無改修

2. **ブランチで隔離 → 巻き戻し容易**
   - `claude-telegram-test`: `hackathon-kachaka`
   - `KSstackchan`: `hackathon-kachaka`
   - `kachaka-hackason`: リポ全体がハッカソン専用、丸ごと破棄可

3. **既存ログを利用**
   - tts-server.log の "serving N bytes" 行を tail-F して StackChan の発話開始を検知（後に撤廃）

4. **container内処理**
   - 完了監視を `docker exec -d` で detached 実行、SSHは即返る
   - VLM呼び出しも container 内 python3 + urllib（curlは container 未インストール）

5. **メモリ厳しい ESP32-S3 への対応**
   - SPIRAM 4MB allocated by tts_client、追加64KBで余裕
   - DRAM が薄いので screen_client は priority低・stack小（8KB）
   - 起動順を制御することで OOM 回避

---

## ファイル一覧

### `claude-telegram-test/bin/` (hackathon-kachaka ブランチ)
- `kachaka-move.sh` (新規) — 前進/後退/旋回/停止
- `kachaka-status.sh` (新規) — ブリッジ・バッテリー報告
- `kachaka-goto.sh` (新規) — 自律走行 + VLM描写 + 画面送信
- `funnel-watchdog.sh` (編集) — `/screen` パス追加
- `tts-server.py` (編集) — `/screen` POST/GET/clear エンドポイント
- `CLAUDE.md.mac` (編集) — Claude への指示書

### `KSstackchan/firmware/main/` (hackathon-kachaka ブランチ)
- `stackchan/screen_client.h` (新規)
- `stackchan/screen_client.cpp` (新規) — `/screen` polling + JPEG decode + LVGL表示
- `main.cpp` (編集) — `screen_client_start()` を delay起動

### `kachaka-hackason/` (main, リポ全体がハッカソン専用)
- `hackason.md` — 連携仕様書
- `hackason-log-2026-05-16.md` — 本ファイル
- `grab_image.py` — カメラ画像取得サンプル
- `kachaka_front.jpg` — 試し撮り画像
- `kachaka-api/` — 公式リポジトリ clone

---

## 巻き戻し手順

```sh
# ブランチを main に戻す
cd ~/Documents/GitHub/claude-telegram-test
git checkout master
git branch -D hackathon-kachaka

cd ~/Documents/GitHub/KSstackchan
git checkout main
git branch -D hackathon-kachaka

# kachaka-hackason リポ全体破棄
rm -rf ~/Documents/GitHub/kachaka-hackason

# Docker
docker stop ros2_bridge-ros2_bridge-1
docker rm ros2_bridge-ros2_bridge-1
docker rmi kachaka-ros2-client
docker rmi <official-bridge-image>

# StackChan firmware を元に戻す → 元ブランチで再ビルド + フラッシュ
# (NVS は保持される)

# Tailscale Funnel の /screen パスは funnel-watchdog.sh の master 版が
# 次回起動時に replay するので自動で消える

# ~/bin/ の同期されたファイルを削除
rm ~/bin/kachaka-move.sh ~/bin/kachaka-status.sh ~/bin/kachaka-goto.sh
# CLAUDE.md は master の CLAUDE.md.mac を再コピーで戻す
```

---

## 学び

- **段階的に動かす**: 「ブリッジ起動 → ステータス取得 → 単発動作 → 自律走行 → 通知 → 画面表示」と1段ずつ確認
- **既存パターンを真似る**: `photo_telegram.cpp` や `tts_client.cpp` を読んで構造を借りた
- **クロスホスト構成は事前確認**: hostname / Tailscale / SSH鍵 / 非対話PATH など盲点が多い
- **メモリ制約のあるMCUは task 起動順が肝**: 既存サービスの初期化を済ませてから新task投入
- **Docker容器内のDNS制約**: MagicDNS は host-only、container は Tailscale IP直叩き
- **CMakeのキャッシュ**: `file(GLOB)` は configure time なので新規 .cpp 追加時は reconfigure
- **Action と Topic の使い分け**: 完了通知が必要なら Action、fire-and-forget なら Topic

---

## 秘匿情報について

このログから以下は除いてあります:
- Tailscale Mac の Funnel DNS 名・IP
- iPhone テザリングSSID
- Bot username, ユーザーemail
- Google Docs ID
- カチャカ本体のIPアドレス
- StackChan の MAC address
- SSH 公開鍵

実際のコードは `hackathon-kachaka` ブランチに残っていますが、上記の固有情報は元のリポジトリのCLAUDE.md・設定ファイルにのみ存在し、本ドキュメントには記載していません。
