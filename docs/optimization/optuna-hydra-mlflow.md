# Optuna × Hydra × MLflow を組み合わせた実践ワークフロー

Hydra をエントリポイントに据え、Optuna でハイパーパラメータ探索を回しながら MLflow へ結果を残す構成は、機械学習プロジェクトでよく使われる定番パターンです。ここでは実際にコードを書きながら、3 つのライブラリがどのように連携するのかを順を追って確認します。

## 1. 必要なライブラリとディレクトリ構成

1. 依存関係をまとめた `requirements.txt` を用意します。

   ```text
   hydra-core==1.3.2
   optuna==3.6.1
   mlflow==2.12.1
   scikit-learn==1.4.2
   ```

2. プロジェクトの雛形を以下のように整えます。

   ```text
   my_project/
   ├── configs/
   │   └── config.yaml
   ├── scripts/
   │   └── objective.py
   └── main.py
   ```

Hydra は `configs/` 配下の YAML を読み込み、Optuna は `objective.py` に定義した目的関数を繰り返し呼び出し、MLflow は `main.py` からログを受け取る、という役割分担を意識すると設計がしやすくなります。

## 2. Hydra で設定を管理する

`configs/config.yaml` では、データセットやモデル、Optuna の探索条件を整理します。Hydra の設定は辞書風に書けるので、試行ごとに上書きしたい値をまとめておきます。

```yaml
dataset:
  test_size: 0.2
  random_state: 42

model:
  class_path: sklearn.ensemble.RandomForestClassifier
  n_estimators: 100
  max_depth: null

optuna:
  direction: maximize
  n_trials: 10
  timeout: null

mlflow:
  tracking_uri: file:./mlruns
  experiment_name: hydra-optuna-tutorial
```

Hydra の `DictConfig` は基本的に読み取り専用です。Optuna の試行内で値を変更するときは `OmegaConf.to_container(cfg, resolve=True)` で通常の辞書に変換したうえでコピーを作成すると安全です。詳しくは [構成・制御 / Hydra](../config/hydra.md) と [OmegaConf 逆引き](../config/omegaconf.md) を参照してください。

## 3. Optuna の目的関数を作る

`scripts/objective.py` に、Hydra の設定を受け取りつつ試行ごとのハイパーパラメータを定義する関数を実装します。ここでは scikit-learn のランダムフォレストを例にします。

```python
from dataclasses import dataclass
from typing import Any, Dict

import mlflow
import optuna
from omegaconf import DictConfig, OmegaConf
from sklearn.datasets import load_breast_cancer
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score
from sklearn.model_selection import train_test_split


@dataclass
class Objective:
    cfg: DictConfig

    def __call__(self, trial: optuna.Trial) -> float:
        # 設定をコピーして安全に書き換えられるようにする
        cfg = OmegaConf.create(OmegaConf.to_container(self.cfg, resolve=True))

        params: Dict[str, Any] = {
            "n_estimators": trial.suggest_int("n_estimators", 50, 300),
            "max_depth": trial.suggest_int("max_depth", 3, 12),
            "min_samples_split": trial.suggest_int("min_samples_split", 2, 10),
        }

        data = load_breast_cancer()
        X_train, X_valid, y_train, y_valid = train_test_split(
            data.data,
            data.target,
            test_size=cfg.dataset.test_size,
            random_state=cfg.dataset.random_state,
        )

        model = RandomForestClassifier(**params, n_jobs=-1, random_state=cfg.dataset.random_state)
        model.fit(X_train, y_train)

        preds = model.predict(X_valid)
        accuracy = accuracy_score(y_valid, preds)

        with mlflow.start_run(run_name=f"trial-{trial.number}", nested=True):
            mlflow.log_params(params)
            mlflow.log_metric("accuracy", accuracy)

        return accuracy
```

Optuna の `Trial` オブジェクトを使って探索空間を宣言し、推定器のスコアを返しています。目的関数内で `mlflow.start_run(..., nested=True)` を使って各試行専用のサブ Run を開いておくと、MLflow 上でも試行結果と Run が 1 対 1 で紐づきます。

## 4. Hydra をエントリポイントにした実行スクリプト

続いて `main.py` を用意し、Hydra から設定を読み込みつつ Optuna と MLflow を初期化します。

```python
from pathlib import Path

import hydra
import mlflow
import optuna
from omegaconf import DictConfig

from scripts.objective import Objective


@hydra.main(version_base=None, config_path="configs", config_name="config")
def main(cfg: DictConfig) -> None:
    mlflow.set_tracking_uri(cfg.mlflow.tracking_uri)
    mlflow.set_experiment(cfg.mlflow.experiment_name)

    objective = Objective(cfg)

    with mlflow.start_run(run_name="random-forest-search"):
        study = optuna.create_study(direction=cfg.optuna.direction)
        study.optimize(objective, n_trials=cfg.optuna.n_trials, timeout=cfg.optuna.timeout)

        mlflow.log_params(study.best_params)
        mlflow.log_metric("best_value", study.best_value)

        # まとめて振り返るための成果物を保存
        summary_path = Path("study_summary.txt")
        summary_path.write_text(str(study.best_trial))
        mlflow.log_artifact(summary_path)

        print("Best params:", study.best_params)
        print("Best value:", study.best_value)


if __name__ == "__main__":
    main()
```

上記のように、Hydra が設定を管理し、Optuna が探索を、MLflow がログを担当します。親 Run (`random-forest-search`) の中で各試行が自動的にサブ Run として保存され、後から UI で比較できます。

## 5. 実行と確認

1. 事前に MLflow のバックエンドディレクトリを作っておきます。

   ```bash
   mkdir -p mlruns
   ```

2. Hydra スクリプトを実行します。

   ```bash
   python main.py optuna.n_trials=20
   ```

   コマンドライン引数で `optuna.n_trials` を上書きできるのが Hydra の利点です。他にも `model.max_depth=5` のように細かく調整できます。

3. 最適なハイパーパラメータとスコアが標準出力に表示され、`mlruns/` 配下に結果が保存されます。MLflow UI で確認したい場合は次のコマンドを使います。

   ```bash
   mlflow ui --backend-store-uri file:./mlruns
   ```

   ブラウザで <http://127.0.0.1:5000> を開くと、各試行のパラメータとメトリクスを一覧で確認できます。

## 6. よくある実践的な工夫

- **GPU の有無で挙動を切り替える**: PyTorch など GPU 対応ライブラリを使う場合、`torch.cuda.is_available()` で確認し、利用可能なデバイス数に応じて `optuna.Study.optimize(n_jobs=...)` を調整します。
- **成果物をまとめて残す**: 推論結果や可視化をファイルとして保存し、`mlflow.log_artifact` で Run に添付しておくと後から再現しやすくなります。`Path("artifacts")` のようにディレクトリを切っておくと整理が楽です。
- **失敗した試行を可視化する**: Optuna の `Study.trials_dataframe()` を呼び出して DataFrame として保存し、MLflow へアップロードすると、後から試行全体の傾向を分析できます。

このワークフローを使うと「設定を安全に管理する (Hydra)」「探索ロジックをカプセル化する (Optuna)」「結果を一元管理する (MLflow)」の 3 点が明確になり、再現性と可視性を両立したハイパーパラメータ探索を実現できます。
