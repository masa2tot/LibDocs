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

### テンプレート構成の詳細

テンプレートを初回確認するときは、以下のファイル群を中心に読み解くと全体像を掴みやすくなります。

- `config/settings.yaml` — DataRobot 接続情報、LLM プロバイダ、ツール固有の設定値をまとめたメイン設定。環境ごとの上書きは `config/environments/<env>.yaml` で行います。
- `config/prompts/` — システムプロンプトやツール呼び出し時のテンプレート。CrewAI など役割駆動型フレームワークでは、ここを編集することでエージェントの人格や動作方針を調整します。
- `agents/*.py` — エージェントの定義。LangGraph の場合はノード（エージェント）ごとにクラスやファクトリ関数が分かれ、RAG テンプレートでは Retriever・Synthesizer の差し替えが容易な構成です。
- `tasks/*.py` — ワークフローの各タスク。入力・出力のスキーマ、前提条件、タスク完了時に記録する成果物などを定義します。
- `tests/` — pytest ベースの簡易テスト。テンプレートの動作確認や将来的な回帰防止に利用できます。

複数テンプレートを比較する場合は、`templates/shared/` に格納された共通ユーティリティも確認してください。LLM 呼び出しや DataRobot API ラッパーなど、全テンプレートで再利用される処理がまとまっています。

### CLI コマンド チートシート

`pip install -e .` 後に利用できる `dr-agent` CLI では、テンプレートに紐づく一連の操作を統合的に扱えます。よく使うサブコマンドと用途は次の通りです。

| コマンド | 用途 | 主なオプション |
| --- | --- | --- |
| `dr-agent templates list` | 利用可能なテンプレートの一覧表示。 | `--verbose` で詳細（説明・最終更新日）付き出力。 |
| `dr-agent init` | テンプレートの複製。 | `--template <name>`、`--branch <branch>`、`--no-install`（依存インストールを抑止）。 |
| `dr-agent run` | ローカルでのエージェント実行。 | `--env <local|test|prod>`、`--verbose`、`--trace`（LangGraph 用ビジュアルログ出力）。 |
| `dr-agent deploy` | DataRobot への本番デプロイ。 | `--environment-id`、`--package-path`、`--skip-build`。 |
| `dr-agent templates update` | ベーステンプレートの更新を取得。 | `--template-path` を指定して既存プロジェクトに反映。 |

コマンドの振る舞いは `dr-agent --help` で随時確認できます。テンプレートに付属する `scripts/` ディレクトリには、CLI を補完するユーティリティ（例: LangGraph の状態遷移可視化、CrewAI の会話ログ整形）が用意されています。

### 設定とシークレットの管理

DataRobot との連携では、API キーや接続先 URL など秘匿情報の扱いが重要です。テンプレートでは以下のパターンが推奨されています。

1. **`.env` ファイルの活用** — `python-dotenv` が同梱されており、`.env` に定義した値が `config/settings.yaml` の `${VAR_NAME}` 記法で展開されます。`templates/shared/.env.example` を参考に必要なキーを洗い出しましょう。
2. **環境ごとの上書き** — `config/environments/local.yaml` などを追加すると、本番用設定を汚さずにローカル・ステージングの差分を管理できます。`dr-agent run --env test` のように実行時に切り替えます。
3. **DataRobot Secrets Manager** — 本番環境では Secrets Manager に登録したキーを参照する形で、CLI から `dr-agent deploy` 時に紐付けられます。`docs/deploying-agents-secrets.md` に設定手順がまとまっています。

LLM プロバイダが複数ある場合は `config/providers/` 以下に別ファイルとして定義し、`settings.yaml` から `!include` で読み込む構成もサポートされています。プロバイダ切り替えを頻繁に行うチームでは、このパターンを採用すると保守性が向上します。

### カスタムツール実装のベストプラクティス

テンプレートの `tasks/` から呼び出すツール（外部 API や社内 DB など）を追加する際は、以下の手順が推奨されています。

1. `templates/shared/tools/` に新規ファイルを作成し、インターフェースに沿ってクラスまたは関数を実装する。
2. 単体テストを `tests/tools/test_<name>.py` に追加し、例外ケースを含む動作を検証する。
3. `tasks/` で当該ツールをインポートし、必要な引数（API エンドポイントやクエリ条件など）を `config/settings.yaml` から取得するように記述する。
4. ツールが返すデータ構造を pydantic モデルで定義し、エージェント間で受け渡す情報の整合性を確保する。

CrewAI テンプレートでは、ツールは `@tool` デコレータ付きの関数として登録します。LangGraph では、ノードの `invoke` メソッド内で明示的に呼び出す形になっているため、非同期処理を行う場合は `async` 関数として実装し、`await` を忘れないようにしましょう。

### ローカルでのテストとデバッグ

テンプレートには実行時の診断を支援する機能が多数含まれています。

- `dr-agent run --trace` — LangGraph テンプレートでグラフの遷移を HTML として出力。`artifacts/traces/` に保存され、ブラウザで可視化できます。
- `dr-agent run --log-level DEBUG` — CrewAI などでも詳細なログを取得。プロンプトへの挿入値やツール呼び出しパラメータが確認できます。
- `pytest` — `tests/` 配下のユニットテストを実行し、ツールやハンドラの回帰を防ぎます。

エージェントが意図せず停止する場合は、`logs/` ディレクトリに出力される `run-<timestamp>.log` を確認し、LangGraph では `graph.state.dump()` を挿入して状態遷移を追跡する方法も有効です。

## DataRobot 連携のポイント

テンプレートから DataRobot へデプロイする際の主要なステップを整理します。

1. **パッケージング** — `dr-agent package --output dist/my-agent.zip` で依存ファイルをまとめ、DataRobot の Custom Application としてアップロードできる形にします。`pyproject.toml` の `project.optional-dependencies["deploy"]` にデプロイ時のみ必要なモジュールが定義されているため、環境に応じてインストールしてください。
2. **環境設定** — DataRobot 側で作成した GenAI Agent 環境 ID を `config/environments/prod.yaml` に記録し、CLI から `--environment-id` オプションで指定します。
3. **接続テスト** — デプロイ後は DataRobot UI の「Test Run」機能で動作確認し、失敗した場合は CLI で `dr-agent logs pull --deployment-id <id>` を実行してログを取得します。

モデルを差し替えたい場合は、DataRobot 上で LLM エンドポイントを新規作成し、テンプレート側 `config/providers/` の該当ファイルを更新して再デプロイします。既存ジョブに影響を与えないよう、ステージング環境でのリハーサルを挟むと安全です。

### テンプレート別の着目ポイント

#### LangGraph テンプレート

- **`graph.py`** — `StateGraph` によってエージェントの遷移図を定義します。テンプレートでは `builder.add_node()` に各タスクを登録し、`builder.add_edge()` や `add_conditional_edges()` で条件分岐を制御します。状態は `pydantic` モデルとして `state.py` に分離されているため、共有データ構造を一元管理できます。
- **遅延評価ツール** — LangGraph 用テンプレートでは、重いツールを `ToolExecutor` 経由で呼び出す仕組みが標準実装されています。`executor.run_async()` を活用すると、I/O バウンドな処理を非同期化できます。
- **トレーシング統合** — OpenTelemetry のエクスポータが組み込まれており、`config/settings.yaml` で `otel.enabled: true` とすると DataRobot Observability へ直接メトリクスを送信できます。

#### CrewAI テンプレート

- **役割ごとのプロンプト** — `config/prompts/` の `planner.yaml`、`researcher.yaml`、`writer.yaml` などに役割の詳細が定義されています。ステップ数を制御したい場合は `tasks/plan.py` で `max_iter` を調整しましょう。
- **メモリ管理** — `agents/memory.py` によって CrewAI の `MemoryBackedConversation` が初期化されます。履歴の保存期間を変えたい場合は、`window_size` を設定し直すことで対応できます。
- **Slack 連携サンプル** — `tools/slack.py` が含まれており、Slack Webhook を使った通知が容易です。Webhook URL は `.env` または Secrets Manager で管理し、`dr-agent run --env prod` でのみ送るように条件分岐させることも可能です。

#### LlamaIndex テンプレート

- **データインジェスト** — `scripts/ingest.py` でローカルファイルをベクトルストアに取り込みます。`--chunk-size` や `--chunk-overlap` を調整することで、回答の粒度を制御できます。
- **インデックス種別の切り替え** — デフォルトでは `VectorStoreIndex` が使われていますが、`config/settings.yaml` の `index_type` を変更し、`indices/` 以下に該当クラスを用意することで `SummaryIndex` などへ差し替えられます。
- **評価レシピ** — `tests/evals/test_retrieval.py` に `llama-index-evals` を使ったリトリーバの評価例が含まれており、Precision/Recall を自動計測できます。

#### Generic Base テンプレート

- **スタートポイント** — もっとも少ない依存関係で構成され、`agents/base_agent.py` と `tasks/base_task.py` を土台に自由なワークフローを構築できます。
- **追加フレームワークの組み込み** — LangChain や Semantic Kernel を試したい場合は、Generic Base に必要な依存関係を `pyproject.toml` に追記し、`tasks/` からインポートして組み合わせるのが安全です。
- **CI/CD 向け** — 無駄なファイルが少ないため、GitHub Actions での自動ビルドや DataRobot への毎日デプロイといった用途にも適しています。

## チーム開発時の運用 Tips

- **ブランチ戦略** — テンプレート自体の更新とプロジェクト固有コードを混在させないよう、`template-sync` ブランチを設けて upstream からの変更を取り込み、`main`（または `develop`）にマージする方式が推奨されています。
- **自動テスト** — GitHub Actions などの CI で `pytest` と `dr-agent run --env test --check`（Dry Run モード）を実行しておくと、DataRobot への誤デプロイを防げます。
- **ドキュメント整備** — プロンプトの責務やツール接続先を `docs/README.md` にまとめておくと、メンテナンスやオンボーディングがスムーズになります。


## よくあるつまずきと対策

- **テンプレートのバージョンが古い**: `docs/getting-started-updating.md` を参照し、`git pull` または `dr-agent templates update` コマンドで最新化します。
- **LLM プロバイダの認証エラー**: `config/settings.yaml` の環境変数を見直し、API キーが正しく設定されているか確認します。
- **依存パッケージの競合**: 仮想環境を作り直し、`pip install -e .` を再実行します。`requirements-dev.txt` が提供されている場合は開発環境用の追加依存も併せて導入します。

## さらに学ぶためのリンク

- [Getting Started ガイド](https://github.com/datarobot-community/datarobot-agent-templates/blob/main/docs/getting-started.md) — 最初のエージェントワークフローを作るための手順。
- [テンプレート更新ガイド](https://github.com/datarobot-community/datarobot-agent-templates/blob/main/docs/getting-started-updating.md) — 既存プロジェクトへの変更を取り込む方法。
- [DataRobot Agentic Workflows ドキュメント](https://docs.datarobot.com/en/docs/gen-ai/genai-agents/index.html) — プラットフォームの全体像を把握したいときに参照。

DataRobot Agent Templates を活用することで、複雑なマルチエージェント構成でも「設定 → 実装 → デプロイ」を共通の型に乗せて進められます。本ガイドを起点に、プロジェクト要件に合わせたカスタマイズへと発展させてください。
