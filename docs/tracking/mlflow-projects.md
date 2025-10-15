# MLflow Projects を言葉から理解する

## はじめに

MLflow Projects は、機械学習のコード・依存関係・実行方法をセットでまとめるための仕様です。これにより別の人が同じコマンドを打つだけで実験を再現できます。まずはキーワードを確認しましょう。

### 用語ミニ辞書

| 用語 | 意味 | 役割 |
| --- | --- | --- |
| `MLproject` ファイル | プロジェクトの設定を記述する YAML ファイル。 | コマンド、パラメータ、使用する環境を宣言する。 |
| エントリーポイント (Entry Point) | プロジェクト内で実行可能なサブコマンド。 | `mlflow run` で `-e` オプションに指定する対象。 |
| パラメータ | `MLproject` に書かれた変数。 | 実行時に `-P` で値を渡し、コードから参照する。 |
| Conda / Docker 環境 | 依存パッケージを固定するための仕組み。 | どのライブラリの組み合わせで Run したかを再現する。 |
| Project URI | プロジェクトの場所を示す識別子。 | ローカルディレクトリ、Git リポジトリ、S3 などを指定できる。 |
| Backend | 実行基盤を切り替えるオプション。 | ローカル実行のほか、Databricks や Kubernetes を利用可能。 |

## インストール

Projects 機能は MLflow 本体に含まれているため、MLflow をインストールすれば利用できます。実際に実行する際は、採用する環境方式（Conda、virtualenv、Docker など）のツールも合わせて準備してください。

```bash
pip install mlflow
# Conda 方式を使うなら
conda install -y conda
# Docker 方式を使うなら Docker Engine をセットアップ
```

## 最小構成の例

ディレクトリ構成とファイルの意味を、コメント付きで示します。

```text
energy-project/
├── MLproject   # プロジェクト設定（YAML）
├── conda.yaml  # Conda 環境定義。依存パッケージを列挙
└── train.py    # 実行したいスクリプト
```

`MLproject` の中身:

```yaml
name: energy-forecasting
conda_env: conda.yaml  # どの環境を使うか
entry_points:
  train:
    parameters:
      n_estimators: {type: int, default: 100}
      max_depth: {type: int, default: 6}
    command: "python train.py --n_estimators {n_estimators} --max_depth {max_depth}"
```

- `name` はプロジェクトの識別子です。
- `conda_env` は実行時に用意する Conda 環境ファイルを指します。
- `entry_points` に複数のサブコマンドを書けます。ここでは `train` というエントリーポイントを定義しています。
- `parameters` の `type` は受け取る値の型（整数・文字列など）を宣言します。
- `command` は実際に実行されるシェルコマンドで、波かっこ `{}` の箇所にパラメータが埋め込まれます。

実行コマンドは次の通りです。

```bash
mlflow run . -e train -P n_estimators=200 -P max_depth=8
```

- `.` は現在のディレクトリを Project URI として指定しています。
- `-e train` でエントリーポイント `train` を選択します。
- `-P` はパラメータを上書きするオプションです。`MLproject` のデフォルト値と異なる設定を試したいときに使います。

## 主要コンセプトを平易に整理

| コンセプト | 説明 | 使いどころ |
| --- | --- | --- |
| Entry Point | プロジェクトで実行可能なタスク。 | 学習・評価・前処理など用途ごとに分けたいとき。 |
| Parameters | 実行時に与える変数。 | ハイパーパラメータの調整や環境切り替え。 |
| Conda / Docker | ランタイム環境を固定するための宣言。 | 「誰が実行しても同じバージョンで動く」状態を作る。 |
| Project URI | プロジェクトの場所。 | Git のコミットや S3 パスなどリモートからも呼び出せる。 |
| Backend | 実行先を切り替えるフック。 | ローカル実行からクラウドクラスターへの移行。 |

## ベストプラクティスと理由

1. **環境のキャッシュを活用する** – Conda での依存解決は時間がかかるため、`CONDA_PKGS_DIRS` を共有ディレクトリに設定すると二回目以降が高速になります。
2. **エントリーポイントを小さく保つ** – `train`、`evaluate`、`prepare_data` のように役割で分けると、パイプラインを柔軟に組み替えられます。
3. **CI/CD と連携する** – GitHub Actions などで `mlflow run` を実行し、成功した Run ID を成果物に添付すると、レビュー時に結果を確認しやすくなります。
4. **アーティファクトを必ず記録する** – `mlflow.log_artifact` をコードに組み込み、モデルやログを Run に紐づけます。これにより Projects と Tracking の情報が一致します。
5. **Project URI を固定する** – Git リポジトリを URI に使う場合はコミットハッシュを含めることで、常に同じ状態を再現できます。

## 関連リファレンス

- [MLflow Projects — Official Documentation](https://mlflow.org/docs/latest/projects.html)
- [MLproject Specification](https://mlflow.org/docs/latest/projects.html#specifying-an-mlproject-file)
- [CLI — `mlflow run`](https://mlflow.org/docs/latest/cli.html#mlflow-run)
