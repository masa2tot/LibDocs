# Talk to My Docs Agents テンプレート

## 概要

[Talk to My Docs Agents](https://github.com/datarobot-community/talk-to-my-docs-agents) は、CrewAI を使ったマルチエージェント構成と FastAPI バックエンド、React/Vite フロントエンド、Pulumi による IaC をひとつのリポジトリで扱えるアプリケーションテンプレートです。Google Drive や Box、ローカルファイルといった複数のドキュメントソースを横断して会話できるように設計されており、クラウド／オンプレミスの DataRobot 環境に合わせて拡張できます。テンプレートは初期構成を提供するものであり、本番導入に際しては自社要件に合わせた調整が必須です。

## 想定ユースケース

- 顧客資料や設計書をまとめたストレージ（Google Drive / Box など）にエージェントアクセスさせ、自然言語で検索・要約を行いたい。
- 既存の DataRobot モデルや外部 LLM を組み込みながら、対話体験を迅速に試作したい。
- インフラ構築、バックエンド、フロントエンド、マルチエージェントの各コンポーネントを一括で管理したい。

## クイックスタートと前提ツール

ローカル開発や Codespaces で扱う場合は、あらかじめ以下のツールをインストールします。

- Python 3.11 以上
- Taskfile.dev（タスクランナー）
- uv（Python パッケージマネージャー）
- Node.js（フロントエンド開発用）
- Pulumi（インフラ構成管理）

依存関係をそろえたら、次のコマンドで各コンポーネントのインストールと開発環境の立ち上げを自動化できます。

```bash
task install-all
task infra:deploy-dev
```

バックエンド、エージェント、フロントエンドをまとめて起動する場合は次を利用します。

```bash
task dev-all
```

`.env.template` を `.env` にコピーし、API キーや OAuth クレデンシャルを設定してから `task` コマンドを実行するのが推奨フローです。

## コンポーネント構成

テンプレートは以下のサブプロジェクトで構成され、タスクランナー経由で個別に開発できます。

- `agent_retrieval_agent/`: CrewAI を用いたマルチエージェントのオーケストレーションとドキュメント処理ロジック。
- `core/`: 共通の Python ユーティリティ。
- `frontend_web/`: React + Vite を使ったユーザーインターフェイス。
- `web/`: FastAPI ベースのバックエンド。
- `infra/`: Pulumi によるインフラストラクチャコード。

## LLM の切り替え

標準では DataRobot の LLM Blueprint + LLM Gateway を利用しますが、登録済みモデル（例: NVIDIA NIM）や既存の DataRobot デプロイ、Azure OpenAI など外部プロバイダーを選択する設定も用意されています。`infra/configurations/llm/` 以下の構成ファイルをシンボリックリンクで切り替えるか、環境変数 `INFRA_ENABLE_LLM` に構成ファイル名を指定することで、デフォルトのモデルやタイムアウト、温度などを調整できます。

## OAuth 連携

Google Drive と Box の OAuth アプリケーションを登録し、クライアント ID / シークレットを `.env` に設定することで、ユーザーが認可したドキュメントにアクセスできます。ローカル開発では `http://localhost:5173/oauth/callback` などのリダイレクト URI を追加し、本番では DataRobot カスタムアプリのコールバック URL を追記します。Pulumi でスタックをデプロイすると、Codespaces やローカル開発向けの OAuth プロバイダー設定が自動で反映されます。

## カスタマイズのポイント

- `agent_retrieval_agent/` に新しいツールやエージェントを追加して会話機能を拡張する。
- `frontend_web/` の React コンポーネントを調整し、ブランドに合わせた UI/UX を作り込む。
- `infra/` の Pulumi スタックやシークレット管理を自社インフラに合わせて変更する。

このテンプレートを基に、開発・検証・本番の 3 環境を段階的に整備すると、ドキュメント対話アプリの再現性と保守性を高められます。
