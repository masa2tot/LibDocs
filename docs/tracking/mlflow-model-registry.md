# MLflow Model Registry を言葉から理解する

## はじめに

MLflow Model Registry は、学習済みモデルを「登録 → テスト → 本番公開」と段階的に管理するための仕組みです。社内で品質チェックを共有するときに役立ちますが、初見では英語の用語が多く戸惑うことがあります。以下の辞書から読み始めましょう。

### 用語ミニ辞書

| 用語 | 意味 | どこで使うか |
| --- | --- | --- |
| Registered Model | モデルの論理名。バージョンをまとめる入れ物。 | 「売上予測モデル」といった名前を付ける。 |
| Model Version | Registered Model に紐づく特定の成果物。 | 各 Run から生成されたモデルが該当。 |
| Stage | モデルの状態を表すラベル。`None` / `Staging` / `Production` / `Archived` など。 | リリース前後のチェックに利用。 |
| Transition Request | Stage を変更するときの承認要求。コメントを残せる。 | チームでレビューするときの記録。 |
| Annotation (Comment / Tag) | Model Version へ付けるメモ。 | どのテストを通過したか等を記録。 |
| REST / Python / CLI | Model Registry を操作するための窓口。 | 用途に応じた API を選ぶ。 |

## インストール

Model Registry は MLflow サーバー機能の一部です。Tracking と同じサーバーを利用するため、次の要素を準備します。

1. **バックエンドデータベース** – MySQL や PostgreSQL など。モデル一覧や Stage 情報を保存します。
2. **アーティファクトストア** – モデルファイルの置き場所。S3 / GCS / Azure Blob / NFS など。
3. **サーバー起動** – `mlflow server --serve-artifacts` を付けると UI からモデルをダウンロードできます。
4. **認証とアクセス制御** – Nginx + OAuth2 Proxy などで、誰が閲覧できるかを制限します。モデルには機密情報が含まれるためです。

## クイックスタート

以下の例では、Run で学習したモデルを登録し、`Staging` ステージへ昇格させ、レビュー用のタグを付与しています。

```python
import mlflow
from mlflow.tracking import MlflowClient

client = MlflowClient(tracking_uri="http://localhost:5000")
model_uri = "runs:/<run_id>/model"  # Run のアーティファクトを指す URI

registered_model = mlflow.register_model(model_uri, "energy-forecasting")

client.transition_model_version_stage(
    name="energy-forecasting",
    version=registered_model.version,
    stage="Staging",  # Stage: モデルの状態を表すラベル
)

client.set_model_version_tag(
    name="energy-forecasting",
    version=registered_model.version,
    key="approval",
    value="qa-passed",  # タグ: テストを通過したことをメモ
)
```

## 主要コンセプトの整理

| コンセプト | 説明 | 利点 |
| --- | --- | --- |
| Registered Model | モデルのグループ名。 | 同じ課題のモデルを 1 つの名前でまとめられる。 |
| Model Version | Registered Model の中の個別バージョン。 | Run から生成されたモデルを順番に管理。 |
| Stage | モデルがどの段階にあるかを示すラベル。 | レビュー中か、本番公開済みかをひと目で把握。 |
| Transition Request | Stage 変更時の承認フロー。 | 誰がいつ承認したかをログとして残せる。 |
| Web UI / API | 操作方法。 | UI は手動管理に、API は自動化に向く。 |

## ベストプラクティスとその理由

1. **ステージポリシーを明文化する** – 例: `Staging` へ上げるには自動テストが成功していること、`Production` へ上げるにはビジネスオーナーの承認が必要といったルールを文書化します。曖昧さをなくし、チーム全体で統一した判断ができます。
2. **CI/CD から自動登録する** – モデル学習パイプラインの最後で `mlflow.register_model` を呼び出し、テスト結果に応じて `client.transition_model_version_stage` を実行すると、手作業を減らせます。
3. **監査ログを保存する** – Transition Request のコメントや履歴を定期的にエクスポートしておくと、内部監査で「いつ誰が承認したか」を示せます。
4. **通知を設定する** – Webhook を利用して Slack / Teams に通知を送ると、Stage 変更を見逃しません。特に本番環境への昇格は事前共有が重要です。
5. **廃止モデルを整理する** – `Archived` ステージを活用し、不要になった Model Version を明示的にアーカイブします。ストレージ容量と UI の見通しが良くなります。

## 関連リファレンス

- [MLflow Model Registry — Official Documentation](https://mlflow.org/docs/latest/model-registry.html)
- [Model Registry REST API](https://mlflow.org/docs/latest/rest-api.html#model-registry)
- [Model Registry CLI](https://mlflow.org/docs/latest/cli.html#mlflow-models)
