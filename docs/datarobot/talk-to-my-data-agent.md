# Talk to My Data Agent でデータに話しかける

[Talk to My Data Agent](https://github.com/datarobot-community/talk-to-my-data-agent) は、DataRobot のアプリケーションテンプレートとして提供される会話型データ分析エージェントです。ファイルやスプレッドシート、Snowflake・BigQuery などのデータソースを接続し、自然言語で質問するだけでチャートやテーブル、コード例を提示してくれます。テンプレートは Azure OpenAI（既定では `gpt-4o`）を想定しつつ、DataRobot LLM Gateway や既存の TextGen モデルにも差し替えられる柔軟性を備えています。

## 特長とユースケース

| 特長 | 具体的なメリット |
| --- | --- |
| **会話型 UI** | ビジネス担当者が自然言語で質問し、可視化やコード提案をすぐ得られる。|
| **スケーラブルな構成** | 数千〜数十億行までを想定し、Snowflake・BigQuery といったクラウド DWH にも接続できる。|
| **テンプレートとしての拡張性** | フロントエンド（React / Streamlit）、LLM、データベースを差し替えて自社要件に最適化可能。|
| **DataRobot との統合** | Pulumi を用いて DataRobot 上のデプロイ・監視まで自動構築できる。|

想定シナリオとしては、営業担当が「最新四半期の売上トレンドを教えて」と尋ねたり、アナリストが補助的な SQL / Python コードを生成して分析を加速したりする場面が挙げられます。

## リポジトリ構成と役割

テンプレートは大きく 3 つのロジックに分かれており、責務が明確です。

| ロジック | ディレクトリ例 | 説明 |
| --- | --- | --- |
| AI ロジック | `deployment_*/` | 会話エージェントのモデル定義。DataRobot の Guarded RAG や Azure OpenAI デプロイを扱う。|
| アプリ ロジック | `frontend/`（Streamlit）、`app_frontend/` + `app_backend/`（React + FastAPI）、`utils/` | ユーザー向け UI と補助的なビジネスロジックを提供。|
| オペレーション ロジック | `infra/__main__.py`、`infra/` | Pulumi による DataRobot リソースのプロビジョニングと設定を管理。|

この分離により、LLM の差し替えや UI の修正を独立に進められ、デプロイメントも Pulumi スタックで再現できます。

## セットアップ手順

1. Python 3.10 以上、[uv](https://docs.astral.sh/uv/getting-started/installation/)、[Taskfile.dev](https://taskfile.dev/#/installation)、Node.js 18+、[Pulumi](https://www.pulumi.com/docs/iac/download-install/) を用意します（DataRobot Codespaces では一部省略可能）。
2. リポジトリを取得します。
   ```bash
   git clone https://github.com/datarobot-community/talk-to-my-data-agent.git
   cd talk-to-my-data-agent
   ```
3. ルート直下の `.env.template` を `.env` にリネームし、DataRobot API トークンやエンドポイント、Azure OpenAI の資格情報、`PULUMI_CONFIG_PASSPHRASE` などを入力します。`FRONTEND_TYPE="streamlit"` に変更すると Streamlit 版 UI を利用できます。
4. もっとも手早い起動方法はクイックスタートスクリプトです。
   ```bash
   python quickstart.py YOUR_PROJECT_NAME
   ```
   スクリプトは仮想環境の作成、依存関係のインストール、`.env` の読み込み、Pulumi スタックのセットアップ、`pulumi up` によるデプロイ、完了時のアプリ URL 表示まで自動化します。

Pulumi を手動で操作したい場合は `set_env.sh`（または Windows の `set_env.bat`）で環境変数をロードした後、`pulumi up` を実行します。

## カスタマイズのポイント

### フロントエンドを切り替える

- `.env` の `FRONTEND_TYPE` を `react`（既定）または `streamlit` に設定。
- 変更後は `python quickstart.py` もしくは `source set_env.sh && pulumi up` で再デプロイします。
- React 版は `app_frontend/`（UI）と `app_backend/`（FastAPI）が連携し、Streamlit 版は `frontend/` 以下で完結します。

### 利用する LLM を変更する

- `infra/settings_generative.py` の `LLM` を `LLMs.AZURE_OPENAI_GPT_4_O` から目的のモデルに変更。
- Trial ユーザーは `LLMs.AZURE_OPENAI_GPT_4_O_MINI` を選択し、`.env` の `OPENAI_API_DEPLOYMENT_ID` で Azure 側のデプロイ名を指定します。
- 既存の DataRobot TextGen デプロイを使う場合は `LLMs.DEPLOYED_LLM` を選び、`.env` に `TEXTGEN_REGISTERED_MODEL_ID` または `TEXTGEN_DEPLOYMENT_ID` と `CHAT_MODEL_NAME` を設定します。
- `utils/api.py` の `ALTERNATIVE_LLM_BIG` / `SMALL` でタスク別のモデル細分化も可能です。

### DataRobot LLM Gateway を利用する

- `.env` から `OPENAI_*` を削除またはコメントアウトし、`USE_DATAROBOT_LLM_GATEWAY=true` を指定。
- その後 `pulumi up` を実行すると、DataRobot 側の LLM カタログから選んだモデルで Guarded RAG / LLM Blueprint が構成され、ローカルで鍵を管理せずに済みます。

### データベース接続を切り替える

- 既定では Snowflake 用の認証情報を想定していますが、`.env` に BigQuery の資格情報を記載し、Pulumi スタックを更新することで切り替えられます。
- 資格情報は DataRobot Secrets へ格納され、アプリから安全に参照されます。

## 運用時のベストプラクティス

- `.env` に含まれるシークレットはローカル管理に留め、必要に応じて DataRobot Secrets / Vault と組み合わせる。
- `pulumi login --local` でローカルステートを使うか、チーム共有なら Pulumi Cloud を活用して履歴を残す。
- DataRobot の [Privacy Policy](https://www.datarobot.com/privacy/) を確認し、機密データを扱う場合はアクセス制御を設計する。
- カスタマイズ後は Taskfile や CI/CD から `python quickstart.py` を再実行してデプロイを自動化する。

## トラブルシューティング

| 症状 | 原因 | 対応 |
| --- | --- | --- |
| `pulumi up` で認証エラーが出る | `.env` に API トークンやエンドポイントが設定されていない。 | `.env` を再確認し、`source set_env.sh` で環境変数を適用する。|
| LLM 呼び出しが失敗する | `OPENAI_*` または LLM Gateway の設定が不足している。 | 必要な鍵・エンドポイントを `.env` に追加するか、Gateway の優先順位を見直す。|
| フロントエンドが意図と違う | `FRONTEND_TYPE` の指定が正しく反映されていない。 | `.env` を修正後に `python quickstart.py` で再デプロイする。|

## 参考リンク

- [Talk to My Data Agent リポジトリ](https://github.com/datarobot-community/talk-to-my-data-agent)
- [Talk To My Data Walkthrough](https://docs.datarobot.com/en/docs/get-started/gs-dr5/talk-data-walk.html)
- [DataRobot アプリケーションテンプレート一覧](https://docs.datarobot.com/en/docs/workbench/wb-apps/app-templates/index.html#application-templates)
- [DataRobot LLM Gateway ドキュメント](https://docs.datarobot.com/en/docs/gen-ai/genai-code/dr-llm-gateway.html)
