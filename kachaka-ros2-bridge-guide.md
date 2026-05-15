# kachaka ROS 2 ブリッジ 利用ガイド (非公式)

kachaka 実機を ROS 2 から触るための、自己解決できる利用ガイドです。

**できるようになること**: PC から ROS 2 ブリッジ経由で kachaka に接続し、トピックを読んだり、コマンドを送って動かしたりする。

**一次情報**: 仕様や最新情報は基本的に公式リポジトリ [`pf-robotics/kachaka-api`](https://github.com/pf-robotics/kachaka-api) を参照してください。接続トラブル、仕様差分、既知不具合が疑われる場合は、必要に応じて [Issues](https://github.com/pf-robotics/kachaka-api/issues) も確認してください。このガイドは、ハッカソン/ハンズオン向けに手順とハマりどころを整理した非公式の補助資料です。

**前提**:
- ROS 2 (Humble) の基本コマンド (`ros2 topic`, `ros2 node` 等) を触ったことがある
- Docker / Docker Compose が使える
- kachaka 実機の電源が入っている
- PC と kachaka が同じネットワークにいる

---

## 0. 事前準備チェック

- [ ] 接続したい **kachaka のIPアドレス**を把握している（kachaka アプリで確認可能）
- [ ] その kachaka の **API設定が ON** になっている（kachaka アプリの開発者向け設定）
- [ ] PC と kachaka が同じ Wi-Fi/有線ネットワークにいる
- [ ] `git`, `docker`, `docker compose` がインストール済み

OS別の注意:

- **Linux**: このガイドでは Docker コンテナ利用を前提にする。DDS は **CycloneDDS** 推奨（後述）。
- **macOS (Docker Desktop)**: Docker Desktop の設定で「Host networking」を有効化（Settings → Resources → Network → "Enable host networking"）。DDS は **CycloneDDS** 推奨。
- **Windows**: WSL2 上の Ubuntu からの利用を推奨。DDS は **CycloneDDS** 推奨。

> 💡 kachaka はアップデートのたびに API 設定が OFF に戻ることがあるので、繋がらないときはまずここを疑う。

> ⚠️ ハッカソンやハンズオン会など、複数人が同じネットワークで ROS 2 を使う場では、事故防止のためブリッジ側・クライアント側の両方で **`ROS_LOCALHOST_ONLY=1`** を設定する。このガイドのコマンドはその前提にしている。ROS 2 通信は同一ホスト内に閉じるため、ブリッジコンテナとクライアントコンテナは同じPC上で動かす。

---

## 1. リポジトリ取得とブリッジ起動

```sh
git clone https://github.com/pf-robotics/kachaka-api.git
cd kachaka-api/tools/ros2_bridge
```

ブリッジを CycloneDDS で起動:

```sh
ROS_LOCALHOST_ONLY=1 RMW_IMPLEMENTATION=rmw_cyclonedds_cpp ./start_bridge.sh <kachakaのIP>
# 例: ROS_LOCALHOST_ONLY=1 RMW_IMPLEMENTATION=rmw_cyclonedds_cpp ./start_bridge.sh 192.168.9.230
```

**正常起動の合図** (こんなログが出ればOK):

```
[component_container_mt-1] [INFO] Load Library: libkachaka_grpc_ros2_bridge.so
[component_container_mt-1] [INFO] 'back_camera' bridge starting
[component_container_mt-1] [INFO] 'diagnostics' bridge starting
...
```

このターミナルはブリッジ専用にして起動したままにしておく。別のターミナルで以降の操作をする。

> ℹ️ ROS 2 Humble がホストにネイティブインストール済みで、ブリッジやクライアントを同じ Linux 環境内で完結させるなら FastDDS (Fast RTPS) のままでも構いません。Docker コンテナ前提では、別コンテナ間の SHM まわりで詰まりやすいので、このガイドでは CycloneDDS を標準手順にしています。
> ```sh
> ROS_LOCALHOST_ONLY=1 ./start_bridge.sh <kachakaのIP>
> ```

---

## 2. 接続確認 (topicを見てみる)

別ターミナルを開く。

### 2.1. トピック一覧

```sh
docker exec -t ros2_bridge-ros2_bridge-1 /opt/kachaka/env.sh ros2 topic list
```

`/kachaka/*` で始まるトピックが一覧表示されればOK。

### 2.2. バッテリー残量

```sh
docker exec -t ros2_bridge-ros2_bridge-1 /opt/kachaka/env.sh \
  ros2 topic echo /kachaka/robot_info/battery_state
```

`percentage: 0.93...` のような値が流れてくれば成功。`Ctrl+C` で止める。

> 💡 `percentage` と `power_supply_status` のみ意味のある値。`voltage: -1.0` などは未設定なので無視してOK。

### 2.3. オドメトリ (QoS 注意)

```sh
docker exec -t ros2_bridge-ros2_bridge-1 /opt/kachaka/env.sh \
  ros2 topic echo --once --qos-reliability best_effort /kachaka/odometry/odometry
```

> ⚠️ **`--qos-reliability best_effort` が必須**。kachaka のセンサ系トピックの多くは BEST_EFFORT で publish されているので、これを付けないと `ros2 topic echo` は何も表示しません。

---

## 3. 自分のコンテナから触る

自前のコードを書いたり rviz2 を立ち上げたりするときは、ブリッジコンテナとは別に「自分用」の ROS 2 環境を立てる。

### 3.1. 推奨: CycloneDDS で両側を揃える

**ブリッジをまだ CycloneDDS で起動していない場合は、一度止めて (Ctrl+C) 再起動**:

```sh
ROS_LOCALHOST_ONLY=1 RMW_IMPLEMENTATION=rmw_cyclonedds_cpp ./start_bridge.sh <kachakaのIP>
```

**別ターミナルで、CycloneDDS 入りのクライアントイメージをビルド**:

```sh
docker build -t kachaka-ros2-client -f skills/kachaka-ros2-env-setup/Dockerfile skills/kachaka-ros2-env-setup
```

以降はこのイメージからクライアントコンテナを起動:

```sh
docker run --rm -it \
  --network=host \
  -e RMW_IMPLEMENTATION=rmw_cyclonedds_cpp \
  -e ROS_LOCALHOST_ONLY=1 \
  kachaka-ros2-client
```

入ったコンテナの中で:

```sh
ros2 topic list
ros2 topic echo --qos-reliability best_effort /kachaka/odometry/odometry
```

これが動けば自分用環境の準備完了。

### 3.2. FastDDS のまま使いたい場合

CycloneDDS にしたくない事情があれば、FastDDS のままでも次の **全部** を満たせば動く:

```sh
docker run --rm -it \
  --network=host \
  --ipc=host \
  --user $(id -u):$(id -g) \
  -e ROS_LOCALHOST_ONLY=1 \
  -e HOME=/tmp \
  osrf/ros:humble-desktop-full ...
```

理由は付録A参照。

---

## 4. 動かしてみる

### 4.1. キー操作で動かす (teleop)

クライアントコンテナの中で:

```sh
ros2 run teleop_twist_keyboard teleop_twist_keyboard \
  --ros-args -r cmd_vel:=/kachaka/manual_control/cmd_vel
```

`i` / `j` / `k` / `l` で前進/旋回。`q` / `z` で速度を変える。

> ⚠️ 走らせる前に **周囲の安全確認** を必ずする。

### 4.2. ゴール位置を指定して自律走行

rviz2 やコマンドで `/kachaka/goal_pose` (geometry_msgs/PoseStamped) を publish すると、目標位置まで自律走行する。

### 4.3. rviz2 で可視化

クライアントコンテナの中で:

```sh
rviz2
```

- Fixed Frame を `odom` に
- Add → By Topic → `/kachaka/lidar/scan`, `/kachaka/odometry/odometry`, `/kachaka/mapping/map` などを追加

---

## 5. 主要トピック チートシート

| トピック | 型 | 用途 |
|---|---|---|
| `/kachaka/odometry/odometry` | nav_msgs/Odometry | オドメトリ |
| `/kachaka/wheel_odometry/wheel_odometry` | 同上 | 車輪odom |
| `/kachaka/imu/imu` | sensor_msgs/Imu | IMU |
| `/kachaka/lidar/scan` | sensor_msgs/LaserScan | 2D LiDAR |
| `/kachaka/front_camera/image_raw` | sensor_msgs/Image | 前カメラ |
| `/kachaka/back_camera/image_raw` | sensor_msgs/Image | 後ろカメラ |
| `/kachaka/tof_camera/image_raw` | sensor_msgs/Image | ToFカメラ |
| `/kachaka/mapping/map` | nav_msgs/OccupancyGrid | 地図 |
| `/kachaka/layout/locations/list` | カスタム | 登録位置一覧 |
| `/kachaka/robot_info/battery_state` | sensor_msgs/BatteryState | 残量 |
| `/kachaka/manual_control/cmd_vel` | geometry_msgs/Twist | 手動速度指令 (送信) |
| `/kachaka/goal_pose` | geometry_msgs/PoseStamped | 目標位置 (送信) |
| `/tf`, `/tf_static` | tf2_msgs/TFMessage | フレーム |

**ほぼ全てのセンサ系トピックは Reliability: BEST_EFFORT**。echo するときは `--qos-reliability best_effort` を忘れずに。自分のノードを書くときは `rclcpp::SensorDataQoS()` / `rclpy.qos.qos_profile_sensor_data` を使うと安全。

---

## 6. 困ったときの対処 (FAQ)

### Q1. ブリッジ起動直後に `error #14: failed to connect to all addresses` が出てクラッシュ

```
[ERROR] 'back_camera' error #14: failed to connect to all addresses
[ERROR] process has died ... exit code -11
```

**原因**: kachaka 本体に gRPC で繋がっていない。

切り分け:
```sh
ping <kachakaのIP>            # ネットワーク到達確認
nc -zv <kachakaのIP> 26400    # APIポートが開いているか
```

- ping が通らない → ネットワーク/IP違い
- ping は通るが nc がハング → **API が無効化されている** 可能性大。kachaka アプリで「API設定」を ON に。
- ping は通るが nc が refused → IPがkachaka以外の機器、もしくは別ポート

### Q2. ログに `manual control enabled` が連発する

```
[INFO] [kachaka.manual_control]: manual control enabled
[INFO] [kachaka.manual_control]: manual control enabled
...
```

kachaka の **電源ボタンが押されて一時停止モード** になっている。本体LEDが **黄色点灯** していないか確認。もう一度電源ボタンを押すと通常モードに戻り、ログも止まる。

この状態だと自律走行コマンドは受け付けない（`cmd_vel` の手動制御は可能）。

### Q3. `ros2 topic list` は見えるのに `ros2 topic echo` が無反応

2つの可能性:

1. **QoS 不一致**: `--qos-reliability best_effort` を付け忘れ。kachakaのトピックはほぼ全部 BEST_EFFORT。
   ```sh
   ros2 topic info -v <topic>   # publisher側のQoS確認
   ```
2. **DDS の SHM 問題** (FastDDS の場合): 別コンテナから触っていて、UID/IPC 条件が揃っていない → 一番簡単な対処は **CycloneDDS に揃える** (セクション 3.1)。

### Q4. `ros2 launch` や `rviz2` でエラー: `Failed to create log directory: //.ros/log`

コンテナ内に書き込み可能なホームディレクトリが無い。`-e HOME=/tmp` を `docker run` に追加。

### Q5. 複数台の kachaka が同じネットワークにいて、トピックが混ざる

ブリッジ起動時に `ROS_DOMAIN_ID` を分ける:

```sh
export ROS_DOMAIN_ID=1   # 各kachakaで異なる値を割り当てる
ROS_LOCALHOST_ONLY=1 RMW_IMPLEMENTATION=rmw_cyclonedds_cpp ./start_bridge.sh <IP>
```

クライアント側でも同じ `ROS_DOMAIN_ID` を `-e` で渡す。ハッカソンやハンズオン会では、あわせて `ROS_LOCALHOST_ONLY=1` も必ず渡す。

### Q6. macOS で接続できない

- Docker Desktop の Host networking を有効化したか確認
- **CycloneDDS を使う**（FastDDS は同一ホスト最適化が前提で macOS の VM 越しでは不安定）
- それでもダメなら、別の Linux マシンや WSL2 から試す

---

## 7. 参考リンク

- 公式リポジトリ: https://github.com/pf-robotics/kachaka-api
- 公式リポジトリの Issues: https://github.com/pf-robotics/kachaka-api/issues
- 公式ドキュメント: `docs/ROS2.md`
- 全機能リスト: `docs/FEATURES.md`
- gRPC 仕様: `docs/GRPC.md`
- Python API: `docs/PYTHON.md`, `docs/kachaka_api_client.ipynb`

---

## 付録A: なぜ CycloneDDS を勧めるのか (中級者向け)

ROS 2 Humble のデフォルト RMW は **FastDDS** で、同一ホスト内では Shared Memory (SHM) transport を優先する。
ブリッジコンテナはホストの UID 1000 で動いており、`/dev/shm` 配下に SHM セグメントを作る。

別の `docker run` コンテナを root で立ち上げると:

- `/dev/shm` を共有していない → そもそも SHM 経由でアクセスできない
- 共有していても UID が違う → 自分が作るセグメントを相手から読めず、双方向確立に失敗

という二段の罠がある。FastDDS のまま使うには次の **全部** が必要:

| 条件 | 役割 |
|---|---|
| `--network=host` | DDS discovery (UDPマルチキャスト) |
| `--ipc=host` または `-v /dev/shm:/dev/shm` | SHMセグメント授受 |
| `--user $(id -u):$(id -g)` | SHMセグメントの所有者を揃える |
| `-e HOME=/tmp` | ROSログ初期化が落ちないように |

実機での検証マトリクス (FastDDS、ブリッジ UID=1000):

| `--network=host` | SHM共有 | `--user 1000:1000` | 結果 |
|---|---|---|---|
| ✓ | – | – | ❌ |
| ✓ | ✓ | – (root) | ❌ |
| ✓ | – | ✓ | ❌ |
| ✓ | ✓ | ✓ | ✅ |

**CycloneDDS** はデフォルトで UDP のみで通信する（SHM は別途 Iceoryx を構成しない限り使わない）ため、これらの罠を全て回避できる。
