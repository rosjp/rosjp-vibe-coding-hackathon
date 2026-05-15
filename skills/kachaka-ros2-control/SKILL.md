---
name: kachaka-ros2-control
description: >
  kachaka を ROS 2 から動かす (teleop, cmd_vel, goal_pose, rviz2) と、自前コンテナから
  接続する際の DDS 設定。Use when the user wants to drive kachaka via teleop, send
  `/kachaka/manual_control/cmd_vel`, publish to `/kachaka/goal_pose` for autonomous
  navigation, visualize in rviz2, run their own ROS 2 container that talks to the bridge,
  configure `RMW_IMPLEMENTATION` (CycloneDDS vs FastDDS), debug FastDDS shared-memory
  (`/dev/shm`) issues across containers, or separate multiple kachakas via `ROS_DOMAIN_ID`.
  キーワード: teleop, cmd_vel, goal_pose, rviz2, CycloneDDS, FastDDS, RMW_IMPLEMENTATION,
  SHM, /dev/shm, --ipc=host, --user, ROS_DOMAIN_ID, マルチロボット.
---

# kachaka を動かす & 自前コンテナから接続する

## TL;DR: 自前コンテナを立てるなら CycloneDDS

**ブリッジ側 (再起動)**:
```sh
RMW_IMPLEMENTATION=rmw_cyclonedds_cpp ./start_bridge.sh <kachakaのIP>
```

**自前のクライアントコンテナ**:
```sh
docker run --rm -it \
  --network=host \
  -e RMW_IMPLEMENTATION=rmw_cyclonedds_cpp \
  osrf/ros:humble-desktop-full \
  bash -c "apt-get update -qq && \
           apt-get install -y -qq ros-humble-rmw-cyclonedds-cpp && \
           source /opt/ros/humble/setup.bash && bash"
```

CycloneDDS は UDP のみ使うため、FastDDS の SHM 問題を回避できる（→ 付録参照）。

## 動かしてみる

### teleop (キー操作)

クライアントコンテナの中で:

```sh
apt-get install -y -qq ros-humble-teleop-twist-keyboard
ros2 run teleop_twist_keyboard teleop_twist_keyboard \
  --ros-args -r cmd_vel:=/kachaka/manual_control/cmd_vel
```

`i` / `j` / `k` / `l` で前進/旋回。`q` / `z` で速度を変える。

> ⚠️ 走らせる前に **周囲の安全確認** を必ずすること。

### 自律走行 (ゴール送信)

`/kachaka/goal_pose` (`geometry_msgs/PoseStamped`) を publish すると目標位置まで自律走行。
rviz2 で `2D Goal Pose` で送るのが一番楽。

### rviz2 で可視化

```sh
rviz2
```

- Fixed Frame: `odom`
- Add → By Topic → `/kachaka/lidar/scan`, `/kachaka/odometry/odometry`, `/kachaka/mapping/map` など

## 複数 kachaka が同じネットワークにいる場合

`ROS_DOMAIN_ID` を kachaka ごとに分けて、DDS の混線を防ぐ:

```sh
export ROS_DOMAIN_ID=1   # 各 kachaka で異なる値
RMW_IMPLEMENTATION=rmw_cyclonedds_cpp ./start_bridge.sh <IP>
```

クライアント側でも同じ `ROS_DOMAIN_ID` を `-e` で渡す。

## 付録: FastDDS の SHM 問題 (なぜ CycloneDDS を勧めるか)

ROS 2 Humble のデフォルト RMW は **FastDDS** で、同一ホスト内では Shared Memory (SHM) transport を優先する。
ブリッジコンテナは UID 1000 で動いており、`/dev/shm` 配下に SHM セグメントを作る。

別の `docker run` コンテナを root で立てると:

- `/dev/shm` を共有していない → SHM 経由でアクセス不可
- 共有していても UID が違う → 自分が作るセグメントを相手から読めず、双方向確立に失敗

実機での検証マトリクス (FastDDS、ブリッジ UID=1000):

| `--network=host` | SHM共有 (`--ipc=host` or `-v /dev/shm`) | `--user 1000:1000` | 結果 |
|---|---|---|---|
| ✓ | – | – | ❌ |
| ✓ | ✓ | – (root) | ❌ |
| ✓ | – | ✓ | ❌ |
| ✓ | ✓ | ✓ | ✅ |

FastDDS のままで使う場合の最小コマンド:

```sh
docker run --rm -it \
  --network=host \
  --ipc=host \
  --user $(id -u):$(id -g) \
  -e HOME=/tmp \
  osrf/ros:humble-desktop-full ...
```

**CycloneDDS** はデフォルトで UDP のみで通信する (SHM は別途 Iceoryx を構成しない限り使わない) ため、これらの罠を全て回避できる。両側を CycloneDDS に揃えるのが楽。

## 関連

- ブリッジ起動・接続トラブル → `kachaka-bridge-setup`
- トピック仕様・QoS → `kachaka-ros2-topics`
- 人間向けガイド: 同リポジトリ `kachaka-ros2-bridge-guide.md`
