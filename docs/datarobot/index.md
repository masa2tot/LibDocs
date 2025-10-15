# DataRobot アプリケーションテンプレートガイド

DataRobot コミュニティでは、会話型アプリやドキュメント検索、マルチエージェントワークフローを素早く試作できるテンプレートを公開しています。このセクションでは代表的な 3 つのテンプレートをまとめ、ディレクトリ構成や導入手順、カスタマイズのポイントを日本語で整理しました。テンプレートを比較しながら、自社のユースケースにもっとも近いものを選ぶ際の参考にしてください。

## 掲載テンプレート

- [Talk to My Data Agent](talk-to-my-data-agent.md) — Snowflake や BigQuery と連携して自然言語でデータ分析できる会話型エージェント。
- [Talk to My Docs Agents](talk-to-my-docs-agents.md) — CrewAI ベースで複数ドキュメントソースへ横断的にアクセスするマルチエージェントテンプレート。
- [DataRobot Agent Templates](datarobot-agent-templates.md) — CrewAI や LangGraph など複数フレームワークのエージェントワークフローを提供するテンプレート集。

## よくある疑問

- **テンプレートの違いは？**
  Talk to My Data Agent はデータベース連携と分析支援に特化し、Talk to My Docs Agents は文書検索と要約が中心、DataRobot Agent Templates は複数フレームワークに対応した汎用的なスターターキットです。
- **どのテンプレートから始めればよい？**
  データ分析が主目的なら Talk to My Data Agent、ドキュメントの検索・要約が必要なら Talk to My Docs Agents、既存のフレームワークで自由に組み立てたい場合は DataRobot Agent Templates から着手するとスムーズです。
- **DataRobot 以外の環境でも使える？**
  いずれもローカル開発用の手順が用意されています。Pulumi や Taskfile を活用することで、DataRobot 環境へのデプロイとローカル検証を切り替えながら進められます。

各テンプレートの詳細は個別ページを参照してください。環境構築や認証、LLM 切り替えなどの手順も掲載しています。
