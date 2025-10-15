# Liburary Docs

機械学習研究に必須の公式ライブラリの実用のための要点を日本語で説明するmkdocsです。


以下の技術を説明します。


| レイヤ        | 目的                 | 主要技術                           |
| ---------- | ------------------ | ------------------------------ |
| **構成・制御**  | 実験条件・環境・設定を管理      | Hydra / OmegaConf              |
| **探索・最適化** | ハイパーパラメータ探索        | Optuna / joblib                |
| **記録・再現**  | 実験追跡・可視化           | MLflow                         |
| **学習基盤**   | モデル学習・生成           | PyTorch / NumPy / scikit-learn |
| **データ前処理** | 欠損補完・正規化・外れ値処理     | pandas / sklearn.preprocessing |
| **数理評価**   | 統計指標・Wasserstein距離 | SciPy / NumPy                  |
| **可視化**    | 次元圧縮・分布描画          | Matplotlib / seaborn / UMAP    |
| **ロギング**   | 出力・進捗監視            | logging           |
| **構造的管理**  | CLI運用・環境管理         | uv / venv / hydra-runner       |

---

## 🧭 1. 設定と構成：Hydra & OmegaConf

### 🌊 Hydra

**公式**：[https://hydra.cc/docs/intro/](https://hydra.cc/docs/intro/)

* **役割**：設定の合成・分岐・再現性管理
* **キーポイント**：CLI上での動的上書き (`training.lr=0.001`)
* **内部構造**：

  * `hydra.main()` → `OmegaConf.load()`
  * `overrides` を `merge_with_dotlist()` でマージ
  * 設定を `DictConfig` として安全に注入

**内部クラス**

* `ConfigLoader`（YAML解析）
* `JobRuntime`（出力パス・ログ構造）
* `Launcher`（マルチプロセス実行）

---

### 🧩 OmegaConf

**公式**：[https://omegaconf.readthedocs.io/](https://omegaconf.readthedocs.io/)

Hydraの基盤を支える設定ライブラリ。
Python辞書を「型安全・階層的・動的」に扱うことができます。

```python
from omegaconf import OmegaConf
cfg = OmegaConf.create({"training": {"lr": 0.001}})
print(cfg.training.lr)  # 0.001
```

**特徴**

* `OmegaConf.to_yaml(cfg)` で構造を可視化
* `OmegaConf.merge()` で複数設定統合
* 型アノテーションで構成の誤りを防止

---

## 🔬 2. 最適化と探索：Optuna, Joblib, Ray

### ⚙️ Optuna

**公式**：[https://optuna.org/](https://optuna.org/)

自動ハイパーパラメータ探索の定番。
Define-by-Run設計で、動的な探索空間を生成します。

**内部構造**

| クラス       | 役割                             |
| --------- | ------------------------------ |
| `Study`   | 全体管理・DB記録・統計                   |
| `Trial`   | 各試行の管理                         |
| `Sampler` | 提案アルゴリズム (TPE, Random, CMA-ES) |
| `Pruner`  | 打ち切り戦略 (MedianPrunerなど)        |

**並列化**

* `study.optimize(..., n_jobs=4)`
  → Joblibがプロセスを並列実行
* DBを共有することでマルチノード探索も可能

---

### 🧩 Joblib

**公式**：[https://joblib.readthedocs.io/](https://joblib.readthedocs.io/)

Python標準の `multiprocessing` を安全に抽象化した並列実行基盤。
Optuna内部でも利用されます。

```python
from joblib import Parallel, delayed
Parallel(n_jobs=4)(delayed(run_once)(cfg) for cfg in configs)
```

特徴：

* プロセス間で安全にデータを共有
* pickleを自動最適化
* シングルライン並列（直感的）

---

### ⚡ Ray (optional integration)

**公式**：[https://docs.ray.io/en/latest/tune/](https://docs.ray.io/en/latest/tune/)

大規模並列最適化を支える分散実験基盤。
OptunaやHydraとも連携可能。

---

## 🧠 3. 記録と可視化：MLflow

### 📊 MLflow

**公式**：[https://mlflow.org/docs/latest/index.html](https://mlflow.org/docs/latest/index.html)

**機能**

* `Tracking`：ハイパーパラメータ・メトリクスの記録
* `UI`：実験結果をブラウザで比較
* `Artifacts`：モデル・グラフ・ログ保存
* `Registry`：モデルバージョン管理

```python
with mlflow.start_run(run_name="trial_1"):
    mlflow.log_param("lr", 0.001)
    mlflow.log_metric("r2", 0.95)
```

**内部構造**

| 要素            | 役割                   |
| ------------- | -------------------- |
| BackendStore  | SQLite/Postgresメタデータ |
| ArtifactStore | モデル・出力保存先            |
| Run           | 実験単位（UUIDで識別）        |

---

### 🧩 Matplotlib / seaborn

グラフ可視化ライブラリ。
MLflowでログした後の可視化にも利用。

**Matplotlib**

* 汎用グラフ描画、出版品質
* `plt.savefig("loss.png")` → MLflowログに登録

**seaborn**

* 統計的可視化に強い
* `sns.histplot(df, x="r2", hue="model")`

---

## ⚙️ 4. 学習と生成：PyTorch / scikit-learn

### 🔥 PyTorch

**公式**：[https://pytorch.org/docs/stable/index.html](https://pytorch.org/docs/stable/index.html)

本プロジェクトの「生成拡散モデル（Diffusion）」や「Cross-Domain Attention」の中核。
GPU計算・自動微分を実現します。

```python
import torch
x = torch.randn(32, 128)
y = torch.nn.Linear(128, 1)(x)
```

特徴：

* `torch.autograd` による自動微分
* `torch.cuda` でGPU最適化
* `torch.nn.Module` によるモデル階層設計

---

### 🧩 scikit-learn

**公式**：[https://scikit-learn.org/stable/](https://scikit-learn.org/stable/)

伝統的な回帰・分類モデルをベースラインとして使用。

例：

```python
from sklearn.ensemble import GradientBoostingRegressor
model = GradientBoostingRegressor()
model.fit(X_train, y_train)
```

**機能利用例**

* `StandardScaler`, `SimpleImputer`
* `r2_score`, `mean_squared_error`
* `train_test_split`

---

## 🧮 5. 統計と数学基盤：NumPy / SciPy

### ⚛️ NumPy

**公式**：[https://numpy.org/doc/stable/](https://numpy.org/doc/stable/)

配列操作と線形代数の中核ライブラリ。

* `np.mean`, `np.std`, `np.linalg.norm`
* ベクトル化演算により高速化

---

### 🔬 SciPy

**公式**：[https://docs.scipy.org/doc/scipy/](https://docs.scipy.org/doc/scipy/)

数学・統計・距離関数などを提供。
特に本プロジェクトでは **Wasserstein距離** を利用。

```python
from scipy.stats import wasserstein_distance
dist = wasserstein_distance(source, target)
```

用途：

* 分布差評価
* ドメイン適応効果の定量比較

---

## 🧰 6. データ前処理と管理：pandas / preprocessing

### 🧾 pandas

**公式**：[https://pandas.pydata.org/docs/](https://pandas.pydata.org/docs/)

構造化データを扱う定番。

* `pd.read_csv`, `df.describe()`
* 欠損値処理（`df.fillna()`）
* DataFrameベースの特徴選択

---

### 🧩 sklearn.preprocessing

* `StandardScaler`：標準化
* `MinMaxScaler`：スケーリング
* `SimpleImputer`：欠損値補完

**内部実装**

* `fit()`：統計計算
* `transform()`：適用
* パイプライン化で再現性を保証

---

## 🎨 7. 可視化とレポート：Visualization & Report Generator

### 📊 Visualization

主に `utils/visualization.py` に実装。

* PCA/UMAP による次元圧縮
* Matplotlib ベースの散布図
* 分布比較（ヒストグラム・KDE）

```python
from sklearn.decomposition import PCA
pca = PCA(n_components=2)
reduced = pca.fit_transform(X)
plt.scatter(reduced[:, 0], reduced[:, 1])
```

---

### 📑 Report Generator

`utils/report_generator.py` により、
実験結果をMarkdown/HTML形式で自動整形。

出力例：

```markdown
# Domain Adaptation Benchmark Report
- Ours (CrossAttention): R² = 0.921
- FEHDA: R² = 0.865
- TabDDPM: R² = 0.841
```

---

## 🧱 8. 並列実行と管理：multiprocessing / uv / hydra.launcher

### 🧩 multiprocessing

Python標準の並列処理ライブラリ。
HydraやOptunaの内部で使用されています。

```python
from multiprocessing import Pool
with Pool(4) as p:
    p.map(run_once, configs)
```

---

### ⚙️ uv

**公式**：[https://github.com/astral-sh/uv](https://github.com/astral-sh/uv)

* **高速パッケージ管理＆仮想環境起動**
* `uv run python ...` で即時実行
* pipより数倍高速で依存解決

---

### 💡 hydra.launcher

Hydraの`multirun`機構を制御。

設定例（`conf/config.yaml`）：

```yaml
hydra:
  launcher:
    class: joblib
    params:
      n_jobs: 4
```

→ 実行時に自動的にマルチプロセス展開。

---


## 🧩 10. まとめ：技術の調和としての設計思想

> Hydra が「構造」を、
> Optuna が「探索」を、
> MLflow が「記録」を担う。

それらの下層には、
PyTorch・NumPy・pandas といった**数理的基盤**があり、
その周囲を統計・可視化・報告系ツールが支えます。

この全体構造こそが
**研究の再現性・効率性・可読性・美学的整合性**を保証するものです。

---

## 🔗 公式ドキュメント一覧

| 技術           | 公式URL                                                                                    |
| ------------ | ---------------------------------------------------------------------------------------- |
| Hydra        | [https://hydra.cc/docs/intro/](https://hydra.cc/docs/intro/)                             |
| OmegaConf    | [https://omegaconf.readthedocs.io/](https://omegaconf.readthedocs.io/)                   |
| Optuna       | [https://optuna.org/](https://optuna.org/)                                               |
| MLflow       | [https://mlflow.org/docs/latest/index.html](https://mlflow.org/docs/latest/index.html)   |
| PyTorch      | [https://pytorch.org/docs/stable/index.html](https://pytorch.org/docs/stable/index.html) |
| scikit-learn | [https://scikit-learn.org/stable/](https://scikit-learn.org/stable/)                     |
| NumPy        | [https://numpy.org/doc/stable/](https://numpy.org/doc/stable/)                           |
| SciPy        | [https://docs.scipy.org/doc/scipy/](https://docs.scipy.org/doc/scipy/)                   |
| pandas       | [https://pandas.pydata.org/docs/](https://pandas.pydata.org/docs/)                       |
| joblib       | [https://joblib.readthedocs.io/](https://joblib.readthedocs.io/)                         |
| matplotlib   | [https://matplotlib.org/stable/](https://matplotlib.org/stable/)                         |
| seaborn      | [https://seaborn.pydata.org/](https://seaborn.pydata.org/)                               |
| uv           | [https://github.com/astral-sh/uv](https://github.com/astral-sh/uv)                       |
