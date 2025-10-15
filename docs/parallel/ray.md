# Ray

Ray は、Python アプリケーションを分散スケールさせるためのフレームワークです。Ray Core、Ray Tune、Ray Data などのサブプロジェクトを通じて、並列実行からハイパーパラメータ探索まで一貫した API を提供します。

## 概要

- **主な用途**: 大規模分散実行、ハイパーパラメータ探索、強化学習、サービング
- **アーキテクチャ**: ヘッドノードとワーカーノードで構成されるクラスタ、分散スケジューラがタスクを配布
- **強み**: 単一ノードからクラスタまで同一コードでスケール、豊富なエコシステム (Tune, Serve, Dataset)

## インストール

```bash
pip install "ray[default]"
```

- GPU を使用する場合は `pip install "ray[default]" --extra-index-url https://download.pytorch.org/whl/cu118` など環境に応じて選択
- Kubernetes やクラスタ利用時は公式イメージを推奨

## クイックスタート (Ray Core)

```python
import ray

ray.init()

@ray.remote
def heavy_compute(x: int) -> int:
    return x * x

futures = [heavy_compute.remote(i) for i in range(10)]
results = ray.get(futures)
print(results)
```

- `ray.init()` でローカルモードを起動（クラスタ接続時は `address="auto"` を指定）
- `@ray.remote` デコレータが分散タスクを表し、`.remote()` メソッドで実行
- `ray.get()` で非同期タスクの結果を取得

## API ハイライト

| モジュール | 主要 API | 説明 |
| --- | --- | --- |
| **ray** | `ray.init`, `ray.get`, `ray.put` | 基本的なタスク実行とオブジェクトストア操作 |
| **ray.actor** | `@ray.remote` クラス | ステートフルなアクターを提供 |
| **ray.tune** | `Tuner`, `TuneConfig` | ハイパーパラメータ探索フレームワーク |
| **ray.train** | `ScalingConfig`, `Trainer` | 分散学習の共通インタフェース |
| **ray.data** | `Dataset` | 分散データパイプライン |

## ベストプラクティス

1. **リソース指定**: `@ray.remote(num_cpus=2, num_gpus=1)` でタスクごとの割り当てを明確化
2. **オブジェクトサイズの最適化**: 共有オブジェクトストアの制限を考慮し、`ray.put` の結果を再利用
3. **スケジューラ調整**: `ray.init(local_mode=True)` でデバッグ、`runtime_env` で依存をパッケージ化
4. **監視**: Ray Dashboard (`ray dashboard`) でジョブ状態を可視化
5. **耐障害性**: `max_retries` パラメータで再実行を設定し、Checkpoints を活用

## 関連リファレンス

- 公式ドキュメント: <https://docs.ray.io/en/latest/>
- Ray Core チュートリアル: <https://docs.ray.io/en/latest/ray-core/walkthrough.html>
- Ray Tune ガイド: <https://docs.ray.io/en/latest/tune/index.html>
