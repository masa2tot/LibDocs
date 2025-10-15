# Hydra Launchers をわかりやすく解説

Hydra は設定ファイルを中心にプログラムを実行するフレームワークで、`--multirun` オプションを使うと同じスクリプトを複数パターンで
繰り返し実行できます。並列実行の仕組みはプラグイン「Launcher」が担いますが、名前だけでは役割が分かりにくいので、用語を説明しな
がら導入方法をまとめます。ここでは最も身近な Joblib Launcher を例にします。

## Hydra Launcher に出てくる用語

| 用語 | 説明 |
| --- | --- |
| Hydra | Facebook（現 Meta）が開発した設定管理フレームワーク。Python スクリプトに設定ファイルを読み込ませやすくする。 |
| コンフィグ（設定ファイル） | YAML 形式のテキストファイル。`defaults` セクションで使用するコンポーネントを切り替えられる。 |
| Launcher | Hydra が複数ジョブを実行するときに、どのように並列化するかを決めるプラグイン。Joblib や Ray など複数種類がある。 |
| ジョブ | Hydra が 1 回プログラムを実行する単位。`--multirun` で複数のジョブをまとめて起動できる。 |
| 作業ディレクトリ | 各ジョブの結果やログを保存するフォルダ。Hydra が自動で分けて作成する。 |

## 必要なパッケージのインストール

```bash
pip install hydra-core joblib
pip install hydra-joblib-launcher  # Joblib バックエンドを使うとき
```

- Hydra 本体は `hydra-core` パッケージに含まれています。バージョン 1.3 以上を推奨します。
- Ray や Submitit など別のバックエンドを使いたいときは、それぞれのプラグイン（`hydra-ray-launcher` など）を追加します。

## 最小構成のサンプル

`conf/config.yaml` を次のように作ります。`defaults` で Joblib Launcher を読み込み、並列数を設定しています。

```yaml
defaults:
  - hydra/launcher: joblib

hydra:
  launcher:
    joblib:
      n_jobs: 4  # 同時に走らせるジョブ数
      prefer: processes  # プロセス方式を使う。I/O が多い場合は threading を選べる
```

`main.py` は Hydra のエントリーポイントを用意します。

```python
import hydra
from omegaconf import DictConfig


@hydra.main(config_path="conf", config_name="config", version_base=None)
def main(cfg: DictConfig) -> None:
    print(cfg.task)


if __name__ == "__main__":
    main()
```

コマンドラインから `--multirun` を付けて実行すると、複数のジョブが並列に走ります。

```bash
python main.py --multirun task=alpha,beta,gamma
```

- `task=alpha,beta,gamma` の指定はコンマ区切りで複数値を渡す Hydra の表記です。3 つのジョブが順番にスケジュールされます。
- 実行するとカレントディレクトリに `multirun/` フォルダが作られ、それぞれのジョブに専用サブフォルダが割り当てられます。ログや結果ファ
  イルは自動的に分離されるため、混同しません。

## よく使う API と設定

| コンポーネント | 役割 | ヒント |
| --- | --- | --- |
| `hydra.launcher.Launcher` | すべての Launcher プラグインが継承する抽象クラス。 | 新しい実行基盤を作りたい場合はこのクラスを実装する。 |
| `hydra_plugins.hydra_joblib_launcher.JoblibLauncher` | Joblib を使ってプロセスを生成する既存実装。 | `n_jobs` や `prefer` でプロセス・スレッドを切り替える。 |
| `hydra.utils.instantiate` | コンフィグに書かれたクラスを生成する補助関数。 | Launchers 以外にも、モデルやデータローダの生成で使える。 |
| `hydra.core.hydra_config.HydraConfig` | 実行時のメタ情報（作業ディレクトリなど）を参照できる。 | ログの保存先をプログラム内で確認したいときに便利。 |
| `hydra.multirun` | CLI から複数ジョブを起動するフラグ。 | シングルランに戻すときは `-m` を外して実行する。 |

### Joblib Launcher の代表的な設定例

Hydra で Joblib Launcher を使うときによく見かける設定スニペットを分解してみましょう。

```yaml
_target_: hydra_plugins.hydra_joblib_launcher.joblib_launcher.JoblibLauncher
n_jobs: -1
backend: loky
prefer: processes
```

- `_target_: hydra_plugins.hydra_joblib_launcher.joblib_launcher.JoblibLauncher` – Hydra は `_target_` に指定されたクラスをインスタンス化して Launcher として利用します。ここでは Joblib を使った並列実行を提供する公式実装を選択しています。
- `n_jobs: -1` – 起動するワーカープロセス（またはスレッド）の数です。`-1` に設定すると、利用可能な CPU コアをすべて使って最大限の並列度を確保します。
- `backend: loky` – Joblib の並列化方式を指定します。`loky` はプロセスベースで CPU 集約タスクに適しており、Joblib Launcher で最も一般的に使われるバックエンドです。
- `prefer: processes` – Joblib に対してプロセス方式を優先するよう指示します。各ジョブを独立したプロセスで動かし、メモリ分離を保ちたい場合に便利です。

Hydra の `--multirun` などで複数の実験を同時に走らせる際には、この設定を `conf/hydra/launcher/joblib.yaml` のようなファイルに保存しておくと、マシンのスペックに応じて `n_jobs` だけ調整しながら同じ並列化ポリシーを再利用できます。

## ベストプラクティス

1. **設定ファイルを分割する** – `conf/hydra/launcher/joblib.yaml` のように、Launcher 固有の設定を別ファイルに分けると再利用しやすくなり
   ます。
2. **バックエンドを状況で切り替える** – 開発時は軽量な Joblib Launcher、本番ではクラスタ対応の Ray Launcher など、`defaults` を差し替
   えるだけで切り替えられます。
3. **リソースの上限を把握する** – Joblib Launcher の `n_jobs` は使用するプロセス数です。マシンの CPU コア数を超えると性能が低下する
   ことがあるため、`os.cpu_count()` を参考に設定してください。
4. **成果物ディレクトリを明示する** – `hydra.run.dir` や `hydra.sweep.dir` を設定すると、ジョブの出力先が予測しやすくなります。後で結果
   を分析するときの迷子を防げます。
5. **プラグインごとの制約を確認する** – Submitit Launcher は SLURM のジョブキューと連携するなど、それぞれのプラグインに前提条件があ
   ります。ドキュメントを読み、必要な権限や設定ファイルを揃えてから使用しましょう。

## 参考リンク

- Launcher プラグインの概要: <https://hydra.cc/docs/advanced/launcher_plugins/>
- Joblib Launcher のドキュメント: <https://hydra.cc/docs/plugins/submitit_launcher/#joblib-launcher>
- Hydra 公式リポジトリ: <https://github.com/facebookresearch/hydra/tree/main/plugins>
