# MLflow Model Registry

## 概要

MLflow Model Registry は、モデルのライフサイクル管理（登録、バージョン管理、ステージ遷移、注釈付け）を行うためのリポジトリです。モデルのデプロイフローにおける承認プロセスや監査証跡を提供し、チーム間でのガバナンスを強化します。

- モデルを名前付きエンティティとして登録し、バージョンごとにメタデータを保存
- `Staging` や `Production` などのステージを定義し、リリースゲートを実現
- Web UI・REST API・Python API・CLI から統一的に操作

## インストール

Model Registry は MLflow サーバー機能の一部です。Tracking サーバーと同じバックエンドストアを利用し、DB スキーマの要件を満たす必要があります。

1. バックエンドストアとして MySQL/PostgreSQL などを準備
2. Tracking サーバーを `mlflow server` で起動（`--serve-artifacts` を有効化すると UI 上でモデルダウンロードが可能）
3. アクセス制御のためにリバースプロキシで認証を付与

## クイックスタート

モデルの登録とステージ遷移の例:

```python
import mlflow
from mlflow.tracking import MlflowClient

client = MlflowClient(tracking_uri="http://localhost:5000")
model_uri = "runs:/<run_id>/model"

registered_model = mlflow.register_model(model_uri, "energy-forecasting")
client.transition_model_version_stage(
    name="energy-forecasting",
    version=registered_model.version,
    stage="Staging",
)
client.set_model_version_tag(
    name="energy-forecasting",
    version=registered_model.version,
    key="approval",
    value="qa-passed",
)
```

## 主要コンセプト

- **Registered Model**: モデルの論理名。複数バージョンを保持。
- **Model Version**: Run から生成された特定のモデル成果物。
- **Stage**: `None` / `Staging` / `Production` / `Archived` などのライフサイクル状態。
- **Transition Request**: ステージ変更の承認フロー。API でのコメントや承認者設定が可能。
- **Model Registry Web UI**: バージョン比較、リネージ参照、コメント管理ができる UI。

## ベストプラクティス

- **ステージポリシー**: `Staging` への昇格は自動テストパス、`Production` への昇格はビジネス承認といったルールを明文化。
- **自動化**: CI/CD パイプラインで `mlflow models transition` コマンドを用いてステージ更新を自動化。
- **監査ログ**: Transition リクエストとコメントをエクスポートし、監査要件に対応。
- **通知連携**: Webhook を利用して Slack / Teams にステージ遷移を通知。

## 関連リファレンス

- [MLflow Model Registry — Official Documentation](https://mlflow.org/docs/latest/model-registry.html)
- [Model Registry REST API](https://mlflow.org/docs/latest/rest-api.html#model-registry)
- [Model Registry CLI](https://mlflow.org/docs/latest/cli.html#mlflow-models)
