# 構成・制御ガイド

設定ファイルを使って機械学習の実験を管理するとき、キーワードが多すぎて「どの仕組みが何をしてくれるのか」が分かりづらくなりがちです。このガイドでは Hydra と OmegaConf を軸に、設定を安全に組み立てるための基本概念と運用フローを整理します。

## まず押さえたいキーワード

| 用語 | 説明 | よくある悩みと結び付けると | 
| --- | --- | --- |
| 設定合成 (Config Composition) | 複数の YAML を組み合わせて 1 つの設定を作る考え方。Hydra の `defaults` が担う。 | 「学習率だけ変えたい」「GPU 用と CPU 用で設定を分けたい」を安全に切り替える。 |
| コンフィググループ | `optimizer/adam.yaml` のように同じカテゴリの設定を束ねたフォルダ。 | 「同じ系統のバリエーションを 1 箇所にまとめたい」。 |
| Structured Config | dataclass で型付きの設定を定義するやり方。Hydra と OmegaConf が理解できる。 | 「存在しないキーや型ミスを事前に防ぎたい」。 |
| 補間 (Interpolation) | `${optimizer.lr}` のように別の値を参照する構文。 | 「同じ値を何度も書きたくない」「環境変数と合成したい」。 |
| マルチラン | Hydra の `-m` オプションで複数パターンを一括実行する機能。 | 「ハイパーパラメータ探索を雑にでも一気に回したい」。 |

## Hydra と OmegaConf の役割分担

- **Hydra** はアプリケーションのエントリポイントに寄り添い、CLI から設定を切り替えるためのフレームワークです。
- **OmegaConf** は YAML・辞書・dataclass を統一して扱う設定コンテナで、Hydra の内部でも使われています。

Hydra を使うと CLI オーバーライドやマルチランが得意になり、OmegaConf を単体で使うと「設定を読み書きする純粋なライブラリ」が手に入ります。両者は相互補完の関係です。

## 代表的なディレクトリ構成

```
project/
├── conf/
│   ├── config.yaml
│   ├── dataset/
│   │   ├── cifar10.yaml
│   │   └── imagenet.yaml
│   └── optimizer/
│       ├── adam.yaml
│       └── sgd.yaml
└── train.py
```

- `config.yaml` の `defaults` で「初期構成」を宣言します。
- `dataset/` や `optimizer/` のようにグループを作ると、CLI から `dataset=imagenet` のように切り替えられます。
- 実行時に Hydra が `outputs/YYYY-MM-DD/HH-MM-SS/.hydra/` を生成し、使用した設定を残します。

## クイックスタート

1. **ライブラリをインストール**

   ```bash
   pip install hydra-core==1.3.* omegaconf==2.3.*
   ```

2. **設定を定義**

   ```yaml
   # conf/config.yaml
   defaults:
     - optimizer: adam
   seed: 0
   epochs: 10

   # conf/optimizer/adam.yaml
   name: adam
   lr: 0.001
   ```

3. **エントリポイントで読み込む**

   ```python
   from dataclasses import dataclass
   import hydra
   from omegaconf import OmegaConf


   @dataclass
   class OptimConfig:
       name: str = "adam"
       lr: float = 1e-3


   @dataclass
   class TrainConfig:
       seed: int = 0
       epochs: int = 10
       optimizer: OptimConfig = OptimConfig()


   @hydra.main(version_base="1.3", config_name="config", config_path="conf")
   def main(cfg: TrainConfig) -> None:
       print(OmegaConf.to_yaml(cfg))


   if __name__ == "__main__":
       main()
   ```

4. **CLI から上書き**

   ```bash
   python train.py optimizer=sgd optimizer.lr=0.01 epochs=20
   ```

Hydra が `conf/optimizer/sgd.yaml` を読み込み、ドット記法で指定された値を上書きします。どの設定を使ったかは `outputs/` 配下に YAML として残ります。

## よく使う操作とコマンド

| 目的 | コマンド / API | 補足 |
| --- | --- | --- |
| 設定を 1 つだけ差し替える | `python train.py optimizer=sgd` | `sgd.yaml` が存在しないとエラーになるので事前に作成する。 |
| ネストしたキーを上書きする | `python train.py optimizer.lr=0.0005` | ドット記法で深いキーを指定できる。 |
| 複数パラメータを一括実行 | `python train.py -m optimizer=adam,sgd batch_size=32,64` | 直積で組み合わせたディレクトリが `multirun/` に生成される。 |
| 実行先を変えたい | `python train.py hydra.run.dir=.` | 出力ディレクトリを現在のパスに固定する。 |
| 生成された設定を保存 | `OmegaConf.save(cfg, "outputs/config-used.yaml")` | Hydra 実行中にカスタム保存を追加したい場合。 |

## 運用のコツ

1. **バージョンを明示** – Hydra と OmegaConf は互換性のある組み合わせが決まっています。`requirements.txt` でペアを固定するとチーム内の再現性が高まります。
2. **Structured Config を使う** – dataclass で型を付けると、typo や型違いを実行前に気付きやすくなります。IDE の補完も効きます。
3. **`.hydra/` をコミットしない** – 実行ごとに生成される `config.yaml` や `overrides.yaml` はログとして価値がありますが、Git 管理すると履歴が煩雑になります。`.gitignore` に追加しておきましょう。
4. **共通値は補間でまとめる** – `lr: ${optimizer.base_lr}` のように参照を使うと重複記述が減り、更新漏れを防げます。
5. **環境変数のデフォルトを決める** – `${oc.env:API_TOKEN,not-set}` のように後ろにデフォルトを書くと CI で値がなくても落ちません。

## どちらを選べばよい？

| やりたいこと | 適したツール | 理由 |
| --- | --- | --- |
| CLI から設定を切り替えながら実行したい | Hydra | `@hydra.main` と CLI オーバーライドが標準機能。 |
| Python コード内で設定を動的に組み立てたい | OmegaConf | `OmegaConf.create`, `OmegaConf.merge` で辞書感覚に扱える。 |
| dataclass を設定ファイルとして扱いたい | 両方 | OmegaConf が dataclass を解釈し、Hydra で CLI 制御まで担える。 |
| 並列実行を同じ設定から起動したい | Hydra + Launcher プラグイン | Ray や Slurm など実行基盤を切り替えられる。 |

## 次に読むと良いページ

- [Hydra 徹底ガイド](hydra.md) – CLI オーバーライドやマルチラン、ConfigStore の詳細を解説。
- [OmegaConf 逆引き](omegaconf.md) – コンフィグ生成、補間、構造化バリデーションの具体例を掲載。

公式ドキュメントのリンクやトラブルシューティングは各ページにまとめています。困った時は `outputs/.../.hydra/overrides.yaml` を確認し、「どの設定を使ったか」を記録しておくと再現が容易になります。
