# Hydra 徹底ガイド

Hydra は Python アプリケーションを「設定ファイル中心」で運用するためのフレームワークです。CLI からの上書き、複数パターンの一括実行（マルチラン）、実行ディレクトリの自動整理といった機能が標準で備わっています。このページでは、初期セットアップから代表的な API、プラグイン活用までを段階的に紹介します。

## 基本情報

| 項目 | 内容 |
| --- | --- |
| 対応バージョン | `hydra-core 1.3.*`（OmegaConf 2.3.* と組み合わせるのが推奨） |
| インストール | `pip install hydra-core==1.3.*` |
| 主な依存 | OmegaConf、Antlr4 |
| ドキュメント | [https://hydra.cc/docs/intro/](https://hydra.cc/docs/intro/) |

### 典型的なエントリポイント

```python
import hydra
from omegaconf import DictConfig


@hydra.main(version_base="1.3", config_path="conf", config_name="config")
def main(cfg: DictConfig) -> None:
    print(cfg.model.name)


if __name__ == "__main__":
    main()
```

`hydra.main` が設定の読み込みと CLI オーバーライド処理を肩代わりしてくれます。返ってくる `cfg` は OmegaConf の `DictConfig`（辞書風のオブジェクト）です。

## 設定の合成と CLI オーバーライド

Hydra は `conf/config.yaml` の `defaults` リストを起点に設定を合成します。

```yaml
# conf/config.yaml
defaults:
  - dataset: cifar10
  - optimizer: adam

model:
  name: resnet18
```

```yaml
# conf/optimizer/adam.yaml
name: adam
lr: 0.001
```

CLI からは以下のように差し替えられます。

```bash
python train.py optimizer=sgd optimizer.lr=0.01 dataset=imagenet
```

- `optimizer=sgd` は `conf/optimizer/sgd.yaml` を読み込みます。
- `optimizer.lr=0.01` のようにドット記法でネストした値を上書きできます。
- 不正なキーを指定すると `ConfigCompositionException` などのエラーで気付けます。

### 便利な上書き構文

| パターン | 書き方 | 説明 |
| --- | --- | --- |
| リスト要素の置換 | `dataloader.num_workers=8` | 基本的には辞書のキーと同じ書き方で入れ替えられる。 |
| 追加の設定ファイルをマージ | `+experiment=my-exp` | 先頭に `+` を付けると `defaults` にない設定を追加読み込み。 |
| 削除 | `~optimizer.momentum` | 波線（チルダ）でキーを削除。 |
| 型変換 | `trainer.devices=2` | CLI 文字列を自動的に数値や bool に変換する。 |

## マルチランでパラメータ探索

Hydra では `-m` オプションを付けるだけで複数パターンを直積で実行できます。

```bash
python train.py -m optimizer=adam,sgd optimizer.lr=0.0005,0.001 batch_size=32,64
```

- 実行結果は `multirun/YYYY-MM-DD/HH-MM-SS/` 配下に格納されます。
- 各ランには `.hydra/overrides.yaml` が自動生成され、後から設定を再現できます。
- `-m` と同時に `hydra.sweep.dir` を指定すると出力先をカスタマイズ可能です。

### Sweeper / Launcher プラグイン

| 目的 | プラグイン例 | 使い所 |
| --- | --- | --- |
| 直積以外の探索をしたい | Optuna Sweeper, Nevergrad Sweeper | ベイズ最適化や進化戦略を Hydra から呼び出せる。 |
| 外部ジョブ基盤に投げたい | Submitit Launcher, Ray Launcher | Slurm、Kubernetes、Ray などに対して同じ設定でジョブを実行。 |

プラグインを使うときは `pip install hydra-optuna-sweeper` のように追加インストールし、`conf/hydra/sweeper/optuna.yaml` などの設定を作成します。

## ConfigStore と Structured Config

Python コード上で設定を登録し、CLI から選択させたい場合は `ConfigStore` を利用します。

```python
from dataclasses import dataclass
from hydra.core.config_store import ConfigStore


@dataclass
class AdamConfig:
    name: str = "adam"
    lr: float = 1e-3
    weight_decay: float = 0.0


cs = ConfigStore.instance()
cs.store(group="optimizer", name="adam", node=AdamConfig)
```

`cs.store` によって、`optimizer=adam` が YAML を作らなくても選択できるようになります。dataclass を使うことで IDE の補完が効き、型違いの値を CLI から渡したときに `ValidationError` で検知できます。

### Structured Config を読む

- `OmegaConf.structured(AdamConfig)` で dataclass から設定を生成できます。
- `@hydra.main` の引数に dataclass 型を指定すると、自動的に構造化された設定として受け取れます。
- `cfg.optimizer.lr` のようにアクセスすると float として扱われ、誤った型が渡るとエラーになります。

## 実行ディレクトリとアーティファクトの整理

Hydra は実行のたびに `outputs/DATE/TIME/` を作成し、その中に `.hydra/` ディレクトリを保存します。

| ファイル | 意味 |
| --- | --- |
| `config.yaml` | 実際に使われた最終的な設定。 |
| `overrides.yaml` | CLI から上書きした差分。 |
| `hydra.yaml` | Hydra 自身の挙動に関する設定（出力先など）。 |

`hydra.run.dir=.` や `hydra.job.chdir=False` を指定すれば挙動を変えられますが、まずはデフォルトの自動整理に乗るとログが散らかりません。

## トラブルシューティング

| 症状 | 原因と対処 |
| --- | --- |
| `KeyError: 'optimizer'` | `defaults` に該当グループを入れていない。`- optimizer: adam` を追記する。 |
| `ConfigCompositionException` | 同じグループで複数の設定を読み込んでいる。`defaults` の順序や `+` 記法を確認する。 |
| `HydraConfigException: Config is read-only` | 実行時に `cfg` を直接書き換えている。`OmegaConf.to_container(cfg, resolve=True)` でコピーしてから編集する。 |
| 出力ディレクトリが散らかる | `hydra.run.dir=.` を指定するとカレントディレクトリで実行できるが、基本は `outputs/` を活用し `.gitignore` に登録する。 |

### エラーの全文を取得したい

Hydra の例外では、省略表示されたスタックトレースの末尾に「`Set the environment variable HYDRA_FULL_ERROR=1 for a complete stack trace.`」という案内が必ず付きます。繰り返し表示されて煩わしい場合は、以下のように環境変数を事前に設定しておくと最初からフルスタックトレースが表示され、原因の特定が容易になります。

```bash
# Linux / macOS
export HYDRA_FULL_ERROR=1
python train.py

# Windows (PowerShell)
$Env:HYDRA_FULL_ERROR = "1"
python train.py
```

一時的に切り替えたい場合は、コマンドの前に `HYDRA_FULL_ERROR=1 python train.py` と書いても同じ効果が得られます。設定済みの状態であれば、`object_type=dict` のような Hydra 固有の例外メッセージも元の Python 例外とあわせて確認できます。

## ベストプラクティスまとめ

1. **設定の最小単位を小さく保つ** – `defaults` の 1 行が 1 つの責務を持つように YAML を分割すると差し替えが簡単になります。
2. **共通値は補間で DRY に** – `${model.name}` や `${env:DATA_ROOT,./data}` を活用し、同じ値の重複記述を減らします。
3. **実験ログと設定をセットで保存** – `.hydra/overrides.yaml` を成果物と同じ場所に残すことで、後から再現できます。
4. **プラグインは 1 つずつ追加** – Sweeper や Launcher を組み合わせると設定が複雑になるので、最初は 1 つのプラグインから試すとトラブル時の切り分けが容易です。
5. **型を付けてレビューしやすく** – Structured Config を使うと設定レビュー時に diff が読みやすくなり、意図しない型の変更にすぐ気付けます。

## 設定ファイルの三層構造と運用ポリシー

実践的な Hydra プロジェクトでは、設定ファイルを「土台」「並列制御」「用途別プリセット」の 3 層に分けると再現性と保守性を両立できます。

### 1. 土台：`conf/config.yaml`

- データソース、特徴量／目的変数、学習・生成・拡散条件、モデル、MLflow、Optuna など日常的に共有したい既定値を集約します。
- `defaults` リストで Joblib ランチャ（`hydra/launcher=joblib`）など共通のプラグイン設定も読み込みます。
- 日々の実験ではこのファイルを起点に CLI で細部を上書きし、履歴は `.hydra/overrides.yaml` で追跡します。

### 2. 並列制御：`conf/hydra/launcher/joblib.yaml`

- マルチランのときに重要になる `n_jobs`、バックエンド、スレッド／プロセス優先度を集中管理します。
- サーバーやローカル PC など実行環境ごとに適切な並列度へ差し替えるだけで挙動を安定化できます。
- 他のランチャへ切り替える場合も同じ階層に YAML を追加するだけで済み、責務の分離が保たれます。
- `joblib.yaml` は、Hydra が読み込む「分担作業のルールブック」です。Hydra が全体の献立と段取りを決め、Joblib が CPU コアという複数の
  「料理人」にタスク（ハイパーパラメータースイープ）を振り分けるイメージで運用すると理解しやすくなります。

#### `joblib.yaml` の主要パラメータ

| キー | 役割 | 推奨値と理由 |
| --- | --- | --- |
| `n_jobs: -1` | 使うワーカー数。 | そのマシンで使える CPU コアをすべて使う設定。追加のチューニングなしで探索が最大並列になります。 |
| `backend: loky` | 並列実行の方式。 | CPU 集約タスクに強いプロセスベースの実装。Hydra の Joblib Launcher で最も一般的です。 |
| `prefer: processes` | プロセス／スレッドどちらを優先するか。 | 実験ごとに独立プロセスとして実行し、CPU 競合を避けたいときの定番。 |
| `verbose: 10` | 進捗ログの冗長さ。 | 1 ジョブ完了ごとに 1 行表示され、スイープの進み具合が把握しやすくなります。 |
| `timeout: null` | ジョブの制限時間。 | 長時間かかる訓練でもタイムアウトしないよう制限を外した設定。 |
| `pre_dispatch: "2*n_jobs"` | 事前にキューへ積むタスク数。 | 各ワーカーに余分なタスクを先渡ししてアイドル時間を減らす保守的な値。 |
| `batch_size: auto` | 一括配布するタスク数。 | Joblib に任せて、タスクの粒度に応じた効率的な割り当てを自動調整します。 |

これらの値をテンプレートとしておくと、学習率やバッチサイズなど複数軸の組み合わせをグリッドサーチする際でも、ローカル PC の CPU 資源を
フルに使い切って一度に試行できます。探索規模が変わっても、必要に応じて `n_jobs` だけを環境ごとに上書きすれば、チーム内で共通のルールを
維持しながら高速に実験を回せます。

### 3. 用途別プリセット：`conf/experiment/*.yaml`

- CLI で毎回長いオーバーライドを書かずに済むよう、よく回すスイープ条件を YAML で固定します。
- 例：
  - `architecture_search.yaml` — モデルの次元数、ヘッド数、MLP 層などのアーキテクチャ比較をグリッドで実施。
  - `diffusion_config_search.yaml` — 拡散ハイパーパラメータ（`beta_schedule` や `num_timesteps` など）だけを集中的に検証。
  - `data_size_analysis.yaml` — データ量を変えながら堅牢性を評価。
  - `quick_ablation.yaml` — 小さなグリッドで傾向を素早く確認するスモークテスト。
- Hydra では `python train.py -m +experiment=architecture_search` のように `-m` と `+experiment=...` を併用して呼び出します。

### なぜプリセットを増やすのか

1. **再現性のカプセル化** — どのパラメータを固定し、どれを振るのかを YAML に宣言すると、CLI の生ログよりも差分レビューや再実行が容易になります。
2. **マルチランの省力化** — 実験名を指定するだけで大規模グリッドが 1 コマンドで実行でき、ランチャ設定も必要に応じて上書き可能です。
3. **責務の分離** — 土台／並列制御／用途別プリセットに分けることで、設定の衝突を避けつつ保守しやすくなります。

### どこまで残すかの目安

- **必須**：`config.yaml` と `hydra/launcher/joblib.yaml`。この 2 つだけでも実行は成立します。
- **頻用なら残す**：アーキテクチャ探索やクイックアブレーションなど、開発時に繰り返すスイープ。
- **HPO で代替できるなら整理対象**：Optuna などのベイズ最適化で網羅している軸（例：`num_timesteps`、`beta_schedule`、`target_da_size`）はプリセットを減らしても運用できます。
- 実験プリセットは `experiments/` など専用ディレクトリに整理し、命名規約やタグ付けを決めておくとレビューと自動化がスムーズになります。

## 参考リンク

- [Hydra 公式ドキュメント](https://hydra.cc/docs/intro/)
- [Config Composition ガイド](https://hydra.cc/docs/advanced/defaults_list/)
- [Structured Config チュートリアル](https://hydra.cc/docs/advanced/structured_config/)
- [Launcher / Sweeper 一覧](https://hydra.cc/docs/plugins/overview/)
