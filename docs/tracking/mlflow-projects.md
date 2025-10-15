# MLflow Projects

## 概要

MLflow Projects は、機械学習ワークロードを再現性の高い形式でパッケージ化するための仕様です。`MLproject` ファイルと環境定義（Conda、virtualenv、Docker）を組み合わせて、任意の環境で同じステップを実行可能にします。

- コマンドラインから統一的にパラメータ付きジョブを実行
- 依存関係を `conda.yaml` や Docker イメージで固定
- MLflow Tracking と連携し、Run として結果を保存

## インストール

Projects は MLflow パッケージに同梱されています。追加で必要なツールを用途に応じて準備してください。

- Conda 実行を利用する場合: `conda` または `mamba`
- Docker 実行を利用する場合: Docker Engine

## クイックスタート

プロジェクトディレクトリ構造の例:

```text
energy-project/
├── MLproject
├── conda.yaml
└── train.py
```

`MLproject`:

```yaml
name: energy-forecasting
conda_env: conda.yaml
entry_points:
  train:
    parameters:
      n_estimators: {type: int, default: 100}
      max_depth: {type: int, default: 6}
    command: "python train.py --n_estimators {n_estimators} --max_depth {max_depth}"
```

実行コマンド:

```bash
mlflow run . -e train -P n_estimators=200 -P max_depth=8
```

## 主要コンセプト

- **Entry Point**: プロジェクト内の実行単位。`MLproject` の `entry_points` に定義。
- **Parameters**: CLI から与えられる変数。型とデフォルト値を設定可能。
- **Conda / Docker**: 実行環境を固定するための仕組み。必要に応じて `docker_env` を利用。
- **Project URI**: Git リポジトリ、ローカルディレクトリ、S3 パスなど多様な URI をサポート。
- **Backend**: Databricks や Kubernetes 上でのリモート実行をサポート。`mlflow run` の `-b` オプションで指定。

## ベストプラクティス

- **環境キャッシュ**: Conda 依存解決を高速化するため、`CONDA_PKGS_DIRS` を共有キャッシュに設定。
- **再利用性の高いエントリーポイント**: データ取得や前処理をサブコマンド化し、パイプライン全体をモジュール化。
- **CI/CD 連携**: GitHub Actions や Jenkins から `mlflow run` を呼び出し、Run ID を成果物として記録。
- **アーティファクト管理**: 結果ファイルを `mlflow.log_artifact` でトラッキングし、Projects と Tracking を連携。

## 関連リファレンス

- [MLflow Projects — Official Documentation](https://mlflow.org/docs/latest/projects.html)
- [MLproject Specification](https://mlflow.org/docs/latest/projects.html#specifying-an-mlproject-file)
- [CLI — `mlflow run`](https://mlflow.org/docs/latest/cli.html#mlflow-run)
