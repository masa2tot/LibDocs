# MLflow Tracking を言葉から理解する

## はじめに

MLflow Tracking は、機械学習実験で「いつ・どんな設定で・どんな結果が出たか」を記録する仕組みです。専門用語が多く登場するため、まずはキーワードの意味を押さえてから手順に進みます。

### 用語ミニ辞書

| 用語 | 説明 | どこで登場するか |
| --- | --- | --- |
| メタデータ | データそのものではなく説明情報のこと。例: 学習率、使用した特徴量名。 | Run に付けるパラメータやタグとして保存される。 |
| Run | 1 回の実験記録。パラメータ、評価指標、アーティファクトをまとめて保管する単位。 | `mlflow.start_run()` のブロックで作成。 |
| Experiment | Run を分類するフォルダー。 | `mlflow.set_experiment()` で選択。 |
| アーティファクト | Run の結果として生まれたファイル（モデル、画像、ログなど）。 | `mlflow.log_artifact` で保存。 |
| Tracking サーバー | Run 情報を受け取り保存する常駐アプリ。 | チーム利用時は `mlflow server` コマンドで起動。 |
| Tracking URI | ログの保存先を表す文字列。ローカルディレクトリや HTTP アドレスを指定。 | `mlflow.set_tracking_uri()` で設定。 |

## インストール

Python パッケージを導入するだけで始められます。`[extras]` オプションは SQLAlchemy などサーバー運用に必要な追加依存をまとめて入れる指定です。

```bash
pip install "mlflow[extras]"
```

サーバーを本格運用する場合は、結果を保存するデータベース（SQLite ならファイル 1 つ、PostgreSQL なら専用サーバー）と、モデルなどのファイルを保存するストレージ（ローカルフォルダーや S3）を用意してください。

## クイックスタート

以下は単一の Run を作成し、パラメータ・メトリクス（評価指標）・モデルを記録する最小コードです。コメントで専門用語の意味を補っています。

```python
import mlflow
from sklearn.ensemble import RandomForestRegressor
from sklearn.metrics import mean_squared_error

mlflow.set_tracking_uri("http://localhost:5000")  # Tracking サーバーの場所（Tracking URI）
mlflow.set_experiment("energy_forecasting")       # Experiment: Run をまとめるフォルダー

with mlflow.start_run(run_name="baseline-rf"):
    params = {"n_estimators": 200, "max_depth": 8}  # パラメータ: モデルの設定値
    model = RandomForestRegressor(**params).fit(X_train, y_train)

    mlflow.log_params(params)  # パラメータをメタデータとして保存
    rmse = mean_squared_error(y_test, model.predict(X_test), squared=False)
    mlflow.log_metric("rmse", rmse)  # メトリクス: 精度を数値で記録
    mlflow.sklearn.log_model(model, "model")  # アーティファクトとしてモデルを保存
```

## 主要コンセプトを噛み砕いて理解する

| コンセプト | 何を意味するか | 使いどころ |
| --- | --- | --- |
| Experiment | Run をグループ化する論理名。 | プロジェクト、データセット、環境などで分類したいとき。 |
| Run | 実験 1 回分のメモ帳。 | パラメータや指標、成果物をまとめて残す。 |
| Parameters | モデルや前処理に渡した設定値。 | 後から「どの設定で高精度だったか」を探すときに役立つ。 |
| Metrics | 精度や損失など、Run の評価値。 | Run 間の比較やアラート設定に利用。 |
| Tags | 任意のラベル。例: `mlflow.set_tag("git_commit", "abc123")`。 | コードバージョンや担当者を紐づける。 |
| Artifacts | モデルファイル、画像、ログ。 | Run を再現するために必要な副産物を保存。 |
| Autologging | ライブラリごとに用意された自動記録機能。 | scikit-learn や PyTorch の学習結果を自動で記録したいとき。 |

## ベストプラクティス（理由も添えて解説）

1. **命名規約を決める** – Run 名やタグに規則があると検索しやすくなります。例えば `run_name="train-{git_hash}"` としておくと、後から該当するコードをすぐに見つけられます。
2. **コードのバージョンをタグに残す** – `mlflow.set_tag("git_commit", os.getenv("GIT_COMMIT"))` のように記録しておくと、再実行するときにどのコミットを使えば良いか迷いません。
3. **依存関係をアーティファクトとして保存する** – `requirements.txt` や Conda 環境ファイルを Run に添付すると、別環境で再現するときの指針になります。
4. **アクセス制御を意識する** – `mlflow server` の前に Nginx や OAuth2 Proxy を置くと、社内外のアクセス権限を細かく制御できます。Tracking サーバーには機密データが含まれることが多いためです。
5. **容量を監視する** – Run が増えるとデータベースやストレージが肥大化します。ストレージ使用量をメトリクスとして監視し、古い Run のアーカイブ手順を用意しましょう。

## 関連リファレンス

- [MLflow Tracking — Official Documentation](https://mlflow.org/docs/latest/tracking.html)
- [Tracking API Reference](https://mlflow.org/docs/latest/python_api/mlflow.html#module-mlflow.tracking)
- [MLflow CLI Reference](https://mlflow.org/docs/latest/cli.html)
