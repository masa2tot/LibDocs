# OmegaConf 逆引き

OmegaConf は YAML / JSON / Python オブジェクトを統一的に扱える設定ライブラリです。Hydra の内部コンポーネントとしても使われていますが、単体で利用すると「CLI を持たない純粋な設定管理ツール」として活躍します。このページでは、基礎的な API とよくある課題を逆引き形式で整理します。

## 基本情報

| 項目 | 内容 |
| --- | --- |
| 対応バージョン | `omegaconf 2.3.*`（Hydra 1.3.* と互換） |
| インストール | `pip install omegaconf==2.3.*` |
| 主な機能 | ネスト構造のドットアクセス、補間、環境変数参照、Structured Config との連携 |
| ドキュメント | [https://omegaconf.readthedocs.io/en/latest/](https://omegaconf.readthedocs.io/en/latest/) |

## 使い方の基本

### 辞書やリストから作る

```python
from omegaconf import OmegaConf

cfg = OmegaConf.create({
    "trainer": {
        "devices": 1,
        "precision": 16,
    },
    "optimizer": {
        "name": "adam",
        "lr": 1e-3,
    },
})

print(cfg.optimizer.lr)      # ドット記法
print(cfg["trainer"]["devices"])  # 添字アクセス
```

- `OmegaConf.create` は辞書・リスト・dataclass を受け取り `DictConfig` / `ListConfig` に変換します。
- `OmegaConf.to_yaml(cfg)` で YAML 文字列として出力できます。
- `OmegaConf.to_container(cfg, resolve=True)` で通常の Python オブジェクトに戻せます。

### YAML を読み書きする

```python
cfg = OmegaConf.load("conf/config.yaml")
print(OmegaConf.to_yaml(cfg))
OmegaConf.save(cfg, "conf/generated.yaml")
```

- `.load` はファイルから `DictConfig` を生成します。
- `.save` は YAML 形式でファイルに書き込みます。`resolve=True` を渡すと補間を展開した状態で保存できます。

## 補間と環境変数

```yaml
# conf/config.yaml
work_dir: ${oc.env:WORK_DIR,./outputs}
log_file: ${work_dir}/train.log
``` 

- `${oc.env:VAR,default}` で環境変数 `VAR` を参照しつつ、値が無い場合のデフォルトを指定できます。
- `${work_dir}` のように別のキーを参照するのが補間です。循環参照すると `InterpolationResolutionError` になるので依存関係を整理しましょう。

### 設定をマージする

```python
base = OmegaConf.create({"optimizer": {"name": "adam", "lr": 1e-3}})
override = OmegaConf.create({"optimizer": {"lr": 5e-4}})
merged = OmegaConf.merge(base, override)
print(merged.optimizer.lr)  # 0.0005
```

- `OmegaConf.merge` は後ろの設定で前の値を上書きします。
- リストの場合は結合ではなく置換になるため、必要に応じて `OmegaConf.to_container` で Python リストに変換して扱うと安全です。

## Structured Config と型チェック

OmegaConf は dataclass や TypedDict などの構造化設定を理解できます。

```python
from dataclasses import dataclass
from omegaconf import OmegaConf


@dataclass
class OptimConfig:
    name: str = "adam"
    lr: float = 1e-3


default_cfg = OmegaConf.structured(OptimConfig)
user_cfg = OmegaConf.create({"lr": 1e-4})
merged = OmegaConf.merge(default_cfg, user_cfg)
print(merged.lr)  # 0.0001
```

- `OmegaConf.structured` は dataclass から型付きの設定を生成します。
- 型に合わない値を渡すと `ValidationError` が発生し、早期にバグへ気付けます。
- Hydra と組み合わせると、CLI からの上書きも型チェックされます。

## よくある逆引き

| やりたいこと | 使う API | コツ |
| --- | --- | --- |
| 既存の設定をディープコピーしたい | `OmegaConf.to_container(cfg, resolve=False, structured_config_mode=None)` | `dict` / `list` に戻すことで副作用なく編集できる。 |
| 未定義キーにアクセスしたい | `cfg.get("missing", default)` | 第二引数でデフォルトを渡すと例外を避けられる。 |
| キーの存在を確認したい | `"optimizer" in cfg` | Python の `in` 演算子が使える。 |
| 補間を強制的に評価したい | `OmegaConf.to_container(cfg, resolve=True)` | 補間を展開して値を確認する。 |
| Hydra 実行中の設定を保存したい | `OmegaConf.save(cfg, "outputs/config-used.yaml")` | Hydra が渡す `DictConfig` もそのまま保存できる。 |

## トラブルシューティング

| 症状 | 原因と対処 |
| --- | --- |
| `InterpolationResolutionError` | 参照先が存在しないか循環参照。`OmegaConf.to_yaml(cfg)` で全体を出力して依存関係を確認。 |
| `ValidationError` | Structured Config の型と一致しない値を渡している。`float` フィールドに文字列を渡していないか点検する。 |
| `AttributeError: 'DictConfig' object has no attribute 'xxx'` | ドット記法で存在しないキーを参照。`cfg.get("xxx", default)` を使う。 |
| 環境変数が取得できない | `${oc.env:VAR}` でデフォルトを付け忘れている。CI では `,fallback` を必ず指定する。 |

## ベストプラクティスまとめ

1. **入力データのバリデーションを徹底** – Structured Config を併用し、`OmegaConf.merge` 前に `OmegaConf.to_container` で想定外の値が入っていないか確認します。
2. **設定をテキストで残す** – `OmegaConf.save` を CI や実験スクリプトに組み込み、使用した設定を成果物と一緒に保管します。
3. **補間の依存関係を簡素に** – 深いネストで補間を多用すると循環しやすいので、参照する階層を 2〜3 段までに抑えます。
4. **環境変数アクセスはデフォルト付きで** – `${oc.env:API_TOKEN,missing}` のように後ろへデフォルトを書くと、本番 / CI での設定漏れを防げます。
5. **Hydra との連携を想定** – Hydra を使う予定があるなら `conf/` を階層分けし、OmegaConf 側でも同じフォルダ構造で読み込めるようにしておくと移行がスムーズです。

## 参考リンク

- [OmegaConf チュートリアル](https://omegaconf.readthedocs.io/en/latest/usage.html)
- [Structured Config とバリデーション](https://omegaconf.readthedocs.io/en/latest/structured_config.html)
- [補間の詳細](https://omegaconf.readthedocs.io/en/latest/usage.html#interpolation)
