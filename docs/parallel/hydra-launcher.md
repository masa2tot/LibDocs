# Hydra Launchers

Hydra は設定駆動の実験管理フレームワークであり、`hydra.launcher` プラグインを通じて並列ジョブの実行基盤を切り替えられます。ここでは Joblib Launcher を例に、設定方法と拡張ポイントを解説します。

## 概要

- **主な用途**: Hydra の `multirun` 機能で複数ジョブを並列起動
- **仕組み**: `hydra.launcher` セクションで指定したクラスが `Launchers` API を実装し、ジョブをスケジューリング
- **強み**: 設定ファイルからワーカー数やバックエンドを制御、クラスタ向けプラグイン（Ray、Submitit など）との互換性

## インストール

```bash
pip install hydra-core joblib
pip install hydra-joblib-launcher  # Joblib バックエンドを利用する場合
```

- Ray や Submitit など他のバックエンドは対応するプラグインを追加
- `hydra-core` 1.3 以上を推奨

## クイックスタート

`conf/config.yaml`:

```yaml
defaults:
  - hydra/launcher: joblib

hydra:
  launcher:
    joblib:
      n_jobs: 4
      prefer: processes
```

`main.py`:

```python
import hydra
from omegaconf import DictConfig


@hydra.main(config_path="conf", config_name="config", version_base=None)
def main(cfg: DictConfig) -> None:
    print(cfg.task)

if __name__ == "__main__":
    main()
```

実行:

```bash
python main.py --multirun task=alpha,beta,gamma
```

- `n_jobs` で並列ジョブ数を指定
- Hydra が各ジョブを独立した作業ディレクトリに分離しログを管理

## API ハイライト

| コンポーネント | 説明 |
| --- | --- |
| `hydra.launcher.Launcher` | 全プラグインが実装する抽象基底クラス |
| `hydra_plugins.hydra_joblib_launcher.JoblibLauncher` | Joblib バックエンドを提供 |
| `hydra.utils.instantiate` | コンフィグから Launcher を生成 |
| `hydra.core.hydra_config.HydraConfig` | 実行時の Hydra メタ情報 |
| `hydra.multirun` | CLI でのマルチラン制御フラグ |

## ベストプラクティス

1. **設定の分離**: `conf/hydra/launcher/joblib.yaml` を用意し、上書き設定をモジュール化
2. **バックエンド切り替え**: 開発時は `joblib`、分散実行時は `ray` や `submitit` へ差し替え
3. **リソース制御**: Joblib バックエンドでは `backend` と `prefer` を指定し CPU/IO に応じた戦略を選択
4. **ログ管理**: Hydra の `hydra.run.dir` を明示指定し、成果物の保存先を統一
5. **クラスタ統合**: Submitit Launcher では SLURM、Ray Launcher では Ray クラスタに接続するなど、既存インフラに合わせてプラグインを採用

## 関連リファレンス

- 公式ドキュメント: <https://hydra.cc/docs/advanced/launcher_plugins/>
- Joblib Launcher: <https://hydra.cc/docs/plugins/submitit_launcher/#joblib-launcher>
- Hydra Configs リポジトリ: <https://github.com/facebookresearch/hydra/tree/main/plugins>
