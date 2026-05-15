# kachaka-unofficial-skills (rosjp-vibe-coding-hackathon)

kachaka を ROS 2 から触る作業を支援する、**非公式**のエージェントスキル集です。
ユーザーが「kachaka に繋がらない」「topic が echo できない」「kachaka を動かしたい」といった意図を持ったときに、Claude Code / Codex / Cursor などのエージェントが該当スキルを自動 invoke してその場で答えられるようにします。

## このリポジトリについて

このリポジトリは、ROS Japan UG のイベント
[ROS Japan UG #62 Vibe Codingしばりのハッカソン 〜カチャカ開発編〜](https://rosjp.connpass.com/event/391958/)
で利用するために用意したものです。

イベントでは、カチャカを題材に、AI との対話を中心に実装や調査を進めます。このリポジトリは、参加者が Claude Code / Codex / Cursor などのエージェントに kachaka ROS 2 bridge まわりの知識を渡しやすくするための Agent Skills をまとめています。

基本的な一次情報は Preferred Robotics の [`pf-robotics/kachaka-api`](https://github.com/pf-robotics/kachaka-api) を参照してください。接続トラブル、仕様差分、既知不具合が疑われる場合は、必要に応じて同リポジトリの [Issues](https://github.com/pf-robotics/kachaka-api/issues) も確認してください。このリポジトリのガイドや skills は、ハッカソン/ハンズオン向けに要点と運用上の注意を整理した補助資料です。

## 含まれるスキル

| スキル | カバー範囲 |
|---|---|
| `kachaka-bridge-setup` | ROS 2 ブリッジの起動、接続トラブル、`error #14`、`manual control enabled`、macOS固有設定 |
| `kachaka-ros2-env-setup` | Docker ベースの ROS 2 Humble クライアント環境、CycloneDDS、`ROS_LOCALHOST_ONLY`、同梱 Dockerfile |
| `kachaka-ros2-topics` | トピック一覧、QoS (BEST_EFFORT) の扱い、`ros2 topic echo` 無反応のデバッグ |
| `kachaka-ros2-control` | teleop / cmd_vel / goal_pose / rviz2、`ROS_DOMAIN_ID` |

## 導入方法

### 方法A: `npx skills add` (推奨)

Node.js / npm が使える環境では、[vercel-labs/skills](https://github.com/vercel-labs/skills) の CLI でインストールできます。Claude Code, Codex, Cursor など複数のエージェントに対応しており、実行中の環境に合わせて適切な skills ディレクトリへ配置されます。

```sh
# プロジェクトに追加
npx skills add rosjp/rosjp-vibe-coding-hackathon

# グローバルに追加
npx skills add rosjp/rosjp-vibe-coding-hackathon -g

# 特定スキルだけ追加
npx skills add rosjp/rosjp-vibe-coding-hackathon --skill kachaka-bridge-setup

# 一覧確認
npx skills add rosjp/rosjp-vibe-coding-hackathon --list
```

インストール後、エージェントは `SKILL.md` の `description` を見て、kachaka / ROS 2 bridge / topic / cmd_vel などに関する相談時に必要なスキルを読み込みます。手動で特定スキルだけ入れたい場合は `--skill` を使ってください。

### 方法B: Claude Code 公式マーケットプレイス

```sh
/plugin marketplace add rosjp/rosjp-vibe-coding-hackathon
/plugin install kachaka-unofficial@kachaka-unofficial-skills
```

## リポジトリ構造

```
.
├── .claude-plugin/
│   └── marketplace.json                  # Claude Code marketplace 用 manifest
├── plugins/
│   └── kachaka-unofficial/
│       ├── .claude-plugin/
│       │   └── plugin.json              # Claude Code plugin 用 manifest
│       └── skills -> ../../skills        # symlink (実体は ./skills)
├── skills/                               # canonical (npx skills 用)
│   ├── kachaka-bridge-setup/SKILL.md
│   ├── kachaka-ros2-env-setup/
│   │   ├── SKILL.md
│   │   └── Dockerfile                  # CycloneDDS 入り ROS 2 クライアント環境
│   ├── kachaka-ros2-topics/SKILL.md
│   └── kachaka-ros2-control/SKILL.md
├── kachaka-ros2-bridge-guide.md          # 人間向け参照ガイド
└── README.md
```

各ディレクトリの役割は次の通りです。

- `.claude-plugin/marketplace.json`: Claude Code の plugin marketplace として認識させるための catalog manifest
- `plugins/kachaka-unofficial/`: Claude Code plugin として配布される plugin 本体
- `plugins/kachaka-unofficial/.claude-plugin/plugin.json`: Claude Code plugin の名前・説明・バージョンを定義する manifest
- `plugins/kachaka-unofficial/skills`: Claude Code plugin 規約に合わせた skills 配置。実体は `../../skills` への symlink
- `skills/`: `npx skills add` で直接検出される canonical な Agent Skills 置き場
- `skills/kachaka-ros2-env-setup/Dockerfile`: 自前 ROS 2 クライアント用の Dockerfile。CycloneDDS、`ROS_LOCALHOST_ONLY`、teleop を設定済み
- `kachaka-ros2-bridge-guide.md`: Agent Skills ではなく、人間が通読するための参照ガイド

`skills/` を canonical な置き場所とし、Claude Code の plugin 規約に合わせるために `plugins/kachaka-unofficial/skills` を symlink にしています。実体は1ヶ所のみです。

> ⚠️ **Windows ユーザーへ**: git for windows は symlink を扱えますが、デフォルトで OFF の場合があります。
> `git config --global core.symlinks true` を有効にするか、symlink を解決できない場合は
> `cp -r skills plugins/kachaka-unofficial/` で実体をコピーしてください。

## 人間向けの参照ガイド

`kachaka-ros2-bridge-guide.md` は Agent Skills としてエージェントにインストールするためのファイルではなく、人間が読むためのドキュメントです。
チュートリアル/トラブルシュート集として
[`kachaka-ros2-bridge-guide.md`](./kachaka-ros2-bridge-guide.md) を同梱しています。
スキル経由で得られる情報と内容はほぼ同じですが、最初から最後まで通読する読み物として整形してあります。

## 免責

- 本プラグインは **rosjp-vibe-coding-hackathon** の成果物であり、kachaka 提供元 (Preferred Robotics) の公式プラグインではありません
- kachaka API / ROS 2 bridge の仕様確認は、基本的に公式リポジトリ [`pf-robotics/kachaka-api`](https://github.com/pf-robotics/kachaka-api) を参照してください。必要に応じて [Issues](https://github.com/pf-robotics/kachaka-api/issues) も確認してください
- 動作確認は限られた環境で行われています。実機を動かす操作 (`cmd_vel` 等) は周囲の安全を必ず確認の上で実施してください
- ROS 2 Humble + kachaka API 経由のブリッジ利用が前提です。それ以外の構成 (Python gRPC 直叩き等) はカバーしていません

## ライセンス

MIT License
