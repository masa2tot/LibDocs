# MLflow Tracking

## 概要

MLflow Tracking は、機械学習実験のメタデータ（パラメータ、メトリクス、アーティファクト、ソースコード）を一元管理するコンポーネントです。Web UI・REST API・Python API を通じて Run を管理し、チーム間での共有と再現を容易にします。

- 実験と Run の階層管理で比較が容易
- 任意のストレージ（S3、GCS、Azure Blob、NFS など）をアーティファクトストアとして利用可能
- サーバーモードを利用すれば、複数ユーザー・複数言語から同一のバックエンドにアクセス

## インストール

1. Python パッケージをインストール

   ```bash
   pip install mlflow[extras]
   ```

2. 追跡サーバー用にデータベースとオブジェクトストレージを準備
   - 開発用途: SQLite + ローカルディレクトリ
   - 本番用途: MySQL/PostgreSQL + S3 互換ストレージ

3. CLI でサーバーを起動

   ```bash
   mlflow server \ 
     --host 0.0.0.0 \ 
     --backend-store-uri postgresql://mlflow:pass@db/mlflow \ 
     --default-artifact-root s3://mlflow-artifacts
   ```

## クイックスタート

```python
import mlflow
from sklearn.ensemble import RandomForestRegressor

mlflow.set_tracking_uri("http://localhost:5000")
mlflow.set_experiment("energy_forecasting")

with mlflow.start_run(run_name="baseline-rf"):
    params = {"n_estimators": 200, "max_depth": 8}
    model = RandomForestRegressor(**params).fit(X_train, y_train)

    mlflow.log_params(params)
    mlflow.log_metric("rmse", mean_squared_error(y_test, model.predict(X_test), squared=False))
    mlflow.sklearn.log_model(model, "model")
```

## 主要コンセプト

- **Experiment**: Run の論理的なまとまり。メタデータの命名空間を提供。
- **Run**: 単一実行の記録単位。パラメータ・メトリクス・タグ・アーティファクトを持つ。
- **Tracking URI**: ログ先を指定。`mlruns/`（ローカル）または HTTP(S) エンドポイント。
- **Artifacts**: モデルや可視化ファイルなどの出力。`mlflow.log_artifact` / `mlflow.log_artifacts` で保存。
- **Autologging**: scikit-learn、TensorFlow など主要ライブラリでの自動ロギング機能。

## ベストプラクティス

- **命名規約**: 実験名は `<project>/<feature>/<objective>` の形式で統一し、Run 名はコミットハッシュを含める。
- **バージョン管理**: `mlflow.set_tag("git_commit", os.getenv("GIT_COMMIT"))` のようにコードバージョンをタグ化。
- **依存関係の固定**: `requirements.txt` や Conda 環境をアーティファクトとして保存し、再現性を担保。
- **アクセス制御**: リバースプロキシや認証リバースプロキシ（Nginx + OAuth2 Proxy など）をサーバーの前段に配置。
- **監視**: Run の件数やストレージ使用量をメトリクス化し、容量逼迫を早期検知。

## 関連リファレンス

- [MLflow Tracking — Official Documentation](https://mlflow.org/docs/latest/tracking.html)
- [Tracking API Reference](https://mlflow.org/docs/latest/python_api/mlflow.html#module-mlflow.tracking)
- [MLflow CLI Reference](https://mlflow.org/docs/latest/cli.html)
