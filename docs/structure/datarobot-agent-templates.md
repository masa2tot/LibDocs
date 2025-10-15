# DataRobot Agent Templates 解説

[DataRobot Agent Templates](https://github.com/datarobot-community/datarobot-agent-templates) は、DataRobot 環境でエージェント型ワークフローを素早く構築するためのテンプレート集です。CrewAI や LangGraph といった人気フレームワーク向けに、3 つのエージェントと 3 つのタスクからなるサンプルワークフローを提供し、ローカル開発から本番展開までの初期設定を最小限に抑えられます。本稿では、リポジトリの構成と基本的な使い方を日本語で整理します。

## 収録テンプレートと用途

テンプレートは次の 4 系統に分かれ、いずれも DataRobot 上でのエージェントワークフロー構築を出発点からサポートします。

| テンプレート | 特徴 | 想定シナリオ |
| --- | --- | --- |
| CrewAI | 役割ベースでエージェントを連携させる設計。`crewAI` 本体の用語に沿ったサンプルを完備。 | チーム体制を模した協調型タスクをすぐ試したいとき。 |
| LangGraph | 状態遷移グラフを定義して複雑なフローを制御。LangChain 連携の入り口にもなる。 | 条件分岐や非線形ワークフローを扱いたいとき。 |
| LlamaIndex | RAG（検索拡張生成）向け。ドキュメント検索と回答生成の型を提供。 | 手元データから回答するエージェントを短時間で試作したいとき。 |
| Generic Base | いずれのフレームワークにも依存しない最小構成。 | 自前のマルチエージェント実装や別フレームワークに合わせてカスタマイズする場合。 |

## ディレクトリ構成の概観

リポジトリ直下には、テンプレート共通の設定ファイルと、各フレームワーク用のディレクトリが用意されています。主なパスと役割は以下の通りです。

- `docs/` — Getting Started やテンプレート更新手順など、英語の詳細ドキュメント。
- `templates/<framework>/` — 各フレームワークのサンプルコードと設定。`config/`、`tasks/`、`agents/` フォルダに分かれています。
- `scripts/` — エージェント CLI など、テンプレート操作用のユーティリティスクリプト。
- `pyproject.toml` — 共通の依存関係と CLI 定義。仮想環境にインストールして使います。

実際にテンプレートをカスタマイズする際は、`templates/<framework>/agents/` でエージェントの役割やプロンプトを調整し、`tasks/` でタスクの入出力や実行順序を編集するのが基本的な流れです。

## 初期セットアップ

### リポジトリの取得

クラウド版 DataRobot を利用している場合は、最新版の `main` ブランチをクローンします。オンプレミスの場合は、自身の環境に対応する `release/<バージョン>` ブランチを指定してください。

```bash
# クラウド環境
git clone https://github.com/datarobot-community/datarobot-agent-templates.git

# オンプレミス環境（例: 11.1）
git clone -b release/11.1 https://github.com/datarobot-community/datarobot-agent-templates.git
```

### 依存関係のインストール

テンプレートは Python プロジェクトとして構成されているため、仮想環境を作成したうえで以下のようにインストールします。

```bash
cd datarobot-agent-templates
python -m venv .venv
source .venv/bin/activate
pip install -e .
```

インストール後は、`dr-agent` CLI からテンプレートの作成や実行を操作できます。

## エージェント開発フロー

1. **テンプレートのコピー** — `dr-agent init --template crewai --name my-agent` のように、ベースとなるテンプレートを複製します。
2. **設定の編集** — `config/settings.yaml` で API キーや使用する LLM プロバイダを指定します。`docs/developing-agents-llm-providers.md` には主要プロバイダの設定例が掲載されています。
3. **ツールの追加** — エージェントに外部データベースや API を呼ばせたい場合は、`docs/developing-agents-tools.md` を参考にツールを実装して `tasks/` に登録します。
4. **ローカル検証** — `dr-agent run --template crewai --env local` でローカル実行し、標準出力に出るログを確認します。
5. **DataRobot へのデプロイ** — テンプレート付属のガイド（`docs/getting-started.md`、`docs/developing-agents-cli.md` など）に沿って、DataRobot プラットフォームへ登録します。

## よくあるつまずきと対策

- **テンプレートのバージョンが古い**: `docs/getting-started-updating.md` を参照し、`git pull` または `dr-agent templates update` コマンドで最新化します。
- **LLM プロバイダの認証エラー**: `config/settings.yaml` の環境変数を見直し、API キーが正しく設定されているか確認します。
- **依存パッケージの競合**: 仮想環境を作り直し、`pip install -e .` を再実行します。`requirements-dev.txt` が提供されている場合は開発環境用の追加依存も併せて導入します。

## さらに学ぶためのリンク

- [Getting Started ガイド](https://github.com/datarobot-community/datarobot-agent-templates/blob/main/docs/getting-started.md) — 最初のエージェントワークフローを作るための手順。
- [テンプレート更新ガイド](https://github.com/datarobot-community/datarobot-agent-templates/blob/main/docs/getting-started-updating.md) — 既存プロジェクトへの変更を取り込む方法。
- [DataRobot Agentic Workflows ドキュメント](https://docs.datarobot.com/en/docs/gen-ai/genai-agents/index.html) — プラットフォームの全体像を把握したいときに参照。

DataRobot Agent Templates を活用することで、複雑なマルチエージェント構成でも「設定 → 実装 → デプロイ」を共通の型に乗せて進められます。本ガイドを起点に、プロジェクト要件に合わせたカスタマイズへと発展させてください。
