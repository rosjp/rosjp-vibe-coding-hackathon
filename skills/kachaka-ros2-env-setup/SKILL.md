---
name: kachaka-ros2-env-setup
description: >
  kachaka ROS 2 bridge に接続するための ROS 2 クライアント環境構築。Use when the
  user wants to prepare a Docker-based ROS 2 Humble environment, build the
  `kachaka-ros2-client` image, configure CycloneDDS / ROS_LOCALHOST_ONLY,
  avoid runtime apt-get,
  or set up a container before reading topics, running rviz2, teleop, or custom nodes.
  キーワード: Dockerfile, docker build, docker run, ROS 2 Humble, CycloneDDS,
  RMW_IMPLEMENTATION, ROS_LOCALHOST_ONLY, kachaka-ros2-client, 環境構築, 自前コンテナ.
---

# kachaka ROS 2 クライアント環境セットアップ

## TL;DR: Dockerfile で事前ビルド

この skill には CycloneDDS 入り ROS 2 クライアント用の `Dockerfile` を同梱している。

このリポジトリのルートでビルドする場合:

```sh
docker build -t kachaka-ros2-client -f skills/kachaka-ros2-env-setup/Dockerfile skills/kachaka-ros2-env-setup
```

skill ディレクトリ単体でビルドする場合:

```sh
docker build -t kachaka-ros2-client .
```

クライアントコンテナを起動:

```sh
docker run --rm -it \
  --network=host \
  -e RMW_IMPLEMENTATION=rmw_cyclonedds_cpp \
  -e ROS_LOCALHOST_ONLY=1 \
  kachaka-ros2-client
```

同梱の `Dockerfile` で `ros-humble-rmw-cyclonedds-cpp` と `ros-humble-teleop-twist-keyboard` をビルド時に入れる。実行時に毎回 `apt-get` しない。

## ハッカソン・ハンズオンでの事故防止

複数人が同じネットワークで ROS 2 を使う会場では、DDS discovery が他人の環境と混ざるのを避けるため、ブリッジ側とクライアント側の両方で必ず `ROS_LOCALHOST_ONLY=1` を設定する。

この設定では ROS 2 通信が同一ホスト内に閉じる。ブリッジコンテナとクライアントコンテナは同じPC上で `--network=host` を使って動かす。

## ブリッジ側も CycloneDDS に揃える

`kachaka-api/tools/ros2_bridge` 側のブリッジは次のように起動:

```sh
ROS_LOCALHOST_ONLY=1 RMW_IMPLEMENTATION=rmw_cyclonedds_cpp ./start_bridge.sh <kachakaのIP>
```

CycloneDDS はデフォルトで UDP のみ使うため、FastDDS の SHM 問題を避けやすい。

## FastDDS のままでよいケース

ROS 2 Humble がホストにネイティブインストール済みで、ブリッジやクライアントを同じ Linux 環境内で完結させるなら FastDDS (Fast RTPS) のままでもよい。

Docker コンテナ前提では、別コンテナ間の `/dev/shm` や UID 条件で詰まりやすいので CycloneDDS を推奨する。ハッカソンやハンズオン会では、FastDDS を使う場合でも `ROS_LOCALHOST_ONLY=1` は設定する。

## 関連

- 公式リポジトリ（一次情報）: https://github.com/pf-robotics/kachaka-api
- 必要に応じて公式リポジトリの Issues も確認: https://github.com/pf-robotics/kachaka-api/issues
- ブリッジ起動・接続トラブル → `kachaka-bridge-setup`
- トピック仕様・QoS → `kachaka-ros2-topics`
- teleop / cmd_vel / goal_pose / rviz2 → `kachaka-ros2-control`
- 人間向けガイド: 同リポジトリ `kachaka-ros2-bridge-guide.md`
