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

## ベストプラクティスまとめ

1. **設定の最小単位を小さく保つ** – `defaults` の 1 行が 1 つの責務を持つように YAML を分割すると差し替えが簡単になります。
2. **共通値は補間で DRY に** – `${model.name}` や `${env:DATA_ROOT,./data}` を活用し、同じ値の重複記述を減らします。
3. **実験ログと設定をセットで保存** – `.hydra/overrides.yaml` を成果物と同じ場所に残すことで、後から再現できます。
4. **プラグインは 1 つずつ追加** – Sweeper や Launcher を組み合わせると設定が複雑になるので、最初は 1 つのプラグインから試すとトラブル時の切り分けが容易です。
5. **型を付けてレビューしやすく** – Structured Config を使うと設定レビュー時に diff が読みやすくなり、意図しない型の変更にすぐ気付けます。

## 参考リンク

- [Hydra 公式ドキュメント](https://hydra.cc/docs/intro/)
- [Config Composition ガイド](https://hydra.cc/docs/advanced/defaults_list/)
- [Structured Config チュートリアル](https://hydra.cc/docs/advanced/structured_config/)
- [Launcher / Sweeper 一覧](https://hydra.cc/docs/plugins/overview/)
