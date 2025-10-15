# hydra-runner で実行手順を整理する

`hydra-runner` は Hydra プロジェクトのエントリポイントを簡潔に管理するためのラッパースクリプトです。`python my_app.py` に長い `--config-name` や `+override=value` を付ける代わりに、事前にコマンドを定義しておき、チームメンバーが同じ手順をワンライナーで再実行できるようにします。

## Hydra と hydra-runner の関係

| 要素 | 役割 | 例 |
| --- | --- | --- |
| Hydra | 設定ファイルの階層管理とコマンドラインからのオーバーライド機能を提供するフレームワーク。 | `@hydra.main(config_path="conf", config_name="train")` |
| hydra-runner | Hydra アプリを呼び出す CLI ツール。`hydra-runner run train` のように、設定名やオプションを短く表現できる。 | `hydra-runner --config-name=train --multirun` |
| ランチャー (Launcher) | 実行基盤を差し替える Hydra のプラグイン。 | `hydra-joblib-launcher` や `hydra-ray-launcher` |

`hydra-runner` 自体は Hydra のコアに同梱されており、追加インストールは不要です。`pip install hydra-core` または `uv add hydra-core` で利用できます。

## インストールと設定

```bash
uv init hydra-example
cd hydra-example
uv add hydra-core hydra-joblib-launcher
mkdir -p conf/experiment
cat <<'YAML' > conf/config.yaml
defaults:
  - _self_
  - experiment: base

trainer:
  max_epochs: 5
  accelerator: cpu
YAML

cat <<'YAML' > conf/experiment/base.yaml
optimizer: adam
lr: 0.001
batch_size: 32
YAML
```

アプリケーションコードでは `@hydra.main` デコレーターを利用します。

```python
# src/train.py
import hydra
from omegaconf import DictConfig

@hydra.main(config_path="conf", config_name="config", version_base="1.3")
def run(cfg: DictConfig) -> None:
    print(f"optimizer={cfg.experiment.optimizer}")
    print(f"lr={cfg.experiment.lr}")

if __name__ == "__main__":
    run()
```

## hydra-runner の使い方

`hydra-runner` はプロジェクトルートで次のように利用します。

```bash
# 単発実行
hydra-runner run src.train --config-name config --config-path conf

# 設定を切り替える
hydra-runner run src.train --config-name config +experiment=base
hydra-runner run src.train --config-name config +experiment=large

# 複数条件を一括実行 (マルチラン)
hydra-runner multirun src.train +experiment=base,large
```

ポイントは、Python モジュールパス (`src.train`) を指定すると自動的に仮想環境内の Python で実行される点です。`hydra-runner run` と `hydra-runner multirun` を使い分けることで、単発の検証とパラメータ探索を簡潔に切り替えられます。

## コマンドのテンプレート化

再現性を高めるために、`hydra-runner` のコマンドを Makefile やタスクランナーにまとめておくと便利です。

```makefile
# Makefile
run-base:
	uv run hydra-runner run src.train +experiment=base

run-sweep:
	uv run hydra-runner multirun src.train +experiment=base,large,small
```

`uv run` を前置することで、仮想環境をアクティベートしていない状態でもロックファイルに従った依存関係で実行できます。

## ベストプラクティス

- `conf/` ディレクトリを階層化し、`defaults` リストで組み合わせを制御する。
- 実験ごとに `outputs/` ディレクトリが生成されるため、`.gitignore` で除外し、MLflow などの追跡ツールにアップロードする。
- チームでは `hydra-runner run src.train +experiment=base` のようにコマンドをドキュメント化し、Slack などで共有する。

## トラブルシューティング

| 症状 | 原因 | 対処 |
| --- | --- | --- |
| `ModuleNotFoundError` が発生する | `src/` が Python パスに追加されていない。 | `pyproject.toml` の `[tool.uv.sources]` で `"src"` を指定するか、`uv run` を使う。 |
| `hydra-runner` コマンドが見つからない | `hydra-core` がインストールされていない、または PATH が通っていない。 | `uv sync` 後に `uv run hydra-runner --help` で確認。 |
| `multirun` の結果が 1 件しか出ない | デフォルトで `max_jobs=1` などの制限がある。 | ランチャー設定を見直し、並列度を明示する。 |

## 関連リファレンス

- [Hydra 公式ドキュメント: hydra-runner](https://hydra.cc/docs/advanced/hydra-runner/)
- [hydra-joblib-launcher](https://hydra.cc/docs/plugins/joblib_launcher/)
- [Hydra の設定構造ガイド](https://hydra.cc/docs/tutorials/structured_config/intro/)
