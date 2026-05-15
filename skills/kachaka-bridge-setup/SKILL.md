---
name: kachaka-bridge-setup
description: >
  kachaka ROS 2 ブリッジの起動・接続トラブルシュート。Use when the user is setting up
  the kachaka ROS 2 bridge for the first time, can't connect to kachaka, sees gRPC
  errors like `error #14: failed to connect to all addresses` or `process has died ...
  exit code -11`, sees repeated `manual control enabled` logs, the robot LED is yellow,
  uses kachaka on macOS / Docker Desktop, or mentions `start_bridge.sh`. キーワード:
  kachaka, ROS 2, ブリッジ, 接続, API設定, 一時停止, 黄色LED, Docker Desktop,
  macOS, CycloneDDS, RMW_IMPLEMENTATION, ROS_LOCALHOST_ONLY.
---

# kachaka ROS 2 ブリッジのセットアップとトラブルシュート

## TL;DR (繋がらないとき)

```sh
ping <kachakaのIP>           # L3 OK?
nc -zv <kachakaのIP> 26400   # gRPCポート開いてる?
```

- ping 通らない → IP違い or 別ネットワーク
- ping は通るが nc がハング → **kachaka 本体の「API設定」が OFF** の可能性大（kachaka アプリでON）
- ping は通るが nc が refused → IP は kachaka 以外、もしくは別ポート

## 起動方法

```sh
git clone https://github.com/pf-robotics/kachaka-api.git
cd kachaka-api/tools/ros2_bridge
ROS_LOCALHOST_ONLY=1 RMW_IMPLEMENTATION=rmw_cyclonedds_cpp ./start_bridge.sh <kachakaのIP>
# 例: ROS_LOCALHOST_ONLY=1 RMW_IMPLEMENTATION=rmw_cyclonedds_cpp ./start_bridge.sh 192.168.9.230
```

**正常起動の合図**:

```
[component_container_mt-1] [INFO] Load Library: libkachaka_grpc_ros2_bridge.so
[component_container_mt-1] [INFO] 'back_camera' bridge starting
...
```

この skill では Linux も含めて Docker コンテナ利用を前提に案内する。ブリッジは CycloneDDS で起動するのを推奨。ROS 2 Humble がホストにネイティブインストール済みで、ブリッジやクライアントを同じ Linux 環境内で完結させるなら FastDDS (Fast RTPS) のままでもよい。Docker コンテナ前提では、別コンテナ間の SHM まわりで詰まりやすい。

ハッカソンやハンズオン会では、他参加者の ROS 2 graph と混ざる事故を避けるため、ブリッジ側でもクライアント側でも `ROS_LOCALHOST_ONLY=1` を設定する。

```sh
ROS_LOCALHOST_ONLY=1 RMW_IMPLEMENTATION=rmw_cyclonedds_cpp ./start_bridge.sh <kachakaのIP>
```

macOS の Docker Desktop では、Settings → Resources → Network → "Enable host networking" も必要。

## 事前準備チェック

- [ ] kachaka 本体の **IPアドレス** を把握 (kachaka アプリで確認)
- [ ] **API設定が ON** になっている（kachaka アプリの開発者向け設定）
- [ ] PC と kachaka が同じネットワーク
- [ ] `git`, `docker`, `docker compose` インストール済み

> 💡 kachaka はアップデートのたびに API 設定が OFF に戻ることがある。繋がらないときはまずここを疑う。

## 症状別 対処

### `error #14: failed to connect to all addresses` で起動直後にクラッシュ

```
[ERROR] 'back_camera' error #14: failed to connect to all addresses
[ERROR] 'diagnostics' error #14: failed to connect to all addresses
[ERROR] process has died ... exit code -11
```

gRPC が約 20 秒の接続タイムアウトで失敗 → コンテナごと SIGSEGV (-11) で死亡、というパターン。
本体側 API に到達できていない。上の TL;DR の切り分け手順で確認。

最頻出原因は **kachaka 本体の API設定が OFF**。

### ログに `manual control enabled` が連発する

```
[INFO] [kachaka.manual_control]: manual control enabled
[INFO] [kachaka.manual_control]: manual control enabled
...
```

kachaka の **電源ボタンが押されて一時停止モード** になっている。本体LEDが **黄色点灯** していないか確認。もう一度電源ボタンを押すと通常モードに戻り、ログも止まる。

この状態だと自律走行コマンドは受け付けない（`cmd_vel` の手動制御は可能）。

### macOS で繋がらない

- Docker Desktop の **Host networking を有効化**（Settings → Resources → Network）
- ブリッジを **CycloneDDS** で起動（FastDDS は同一ホスト最適化前提で Docker Desktop の VM 越しは不安定）
  ```sh
  ROS_LOCALHOST_ONLY=1 RMW_IMPLEMENTATION=rmw_cyclonedds_cpp ./start_bridge.sh <IP>
  ```
- それでもダメなら Linux マシン や WSL2 で試す

## 関連

- 公式リポジトリ（一次情報）: https://github.com/pf-robotics/kachaka-api
- 必要に応じて公式リポジトリの Issues も確認: https://github.com/pf-robotics/kachaka-api/issues
- ROS 2 クライアント環境構築 → `kachaka-ros2-env-setup`
- 自分用コンテナから topic を見る/動かす → `kachaka-ros2-topics`, `kachaka-ros2-control`
- 人間向けガイド: 同リポジトリ `kachaka-ros2-bridge-guide.md`
