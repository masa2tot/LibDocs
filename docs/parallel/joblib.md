# Joblib

Joblib は、Python の並列処理と結果キャッシュをシンプルな API で提供するユーティリティライブラリです。公式ドキュメントの章立てに合わせ、基本概念と実践的な活用方法をまとめます。

## 概要

- **主な用途**: 並列マップ、パイプライン計算のメモ化、scikit-learn の内部並列化
- **強み**: `Parallel`/`delayed` による宣言的 API、プロセス・スレッドの両対応、NumPy 配列向けの高速シリアライズ
- **依存関係**: Python 標準の `multiprocessing` を抽象化し安全に扱う

## インストール

```bash
pip install joblib
```

- バージョン 1.3 以降では Python 3.8+ が必要です
- scikit-learn と併用する場合は `pip install scikit-learn` も推奨

## クイックスタート

```python
from joblib import Parallel, delayed


def preprocess(sample: str) -> str:
    return sample.lower()

corpus = ["Data", "Science", "Pipeline"]
results = Parallel(n_jobs=2)(delayed(preprocess)(text) for text in corpus)
print(results)
```

- `Parallel(n_jobs=2)` はワーカー数を指定
- `delayed` でラッピングすることで遅延評価し、ジョブキューに投入

## API ハイライト

| API | 説明 |
| --- | --- |
| `joblib.Parallel` | 並列実行コンテナ。`with` 文にも対応 |
| `joblib.delayed` | 引数を保持したまま遅延呼び出しを生成 |
| `joblib.Memory` | 計算結果をディスクキャッシュ |
| `parallel_backend` | `loky`/`threading`/`multiprocessing` などのバックエンド切り替え |
| `register_parallel_backend` | カスタムバックエンドを登録 |

## ベストプラクティス

1. **バックエンド選択**: CPU バウンドなら `loky` (デフォルト)、I/O バウンドなら `threading`
2. **エラーハンドリング**: `Parallel(..., prefer="processes")` と `BatchingCallback` で詳細ログを取得
3. **キャッシュ活用**: 繰り返し実行される前処理に `Memory` を適用して I/O を削減
4. **scikit-learn 連携**: `n_jobs` パラメータで Joblib 並列化が自動利用される
5. **プロファイリング**: `verbose` オプションでスケジューリングの詳細を観察

## 関連リファレンス

- 公式ドキュメント: <https://joblib.readthedocs.io/>
- 並列計算ガイド: <https://joblib.readthedocs.io/en/latest/parallel.html>
- メモリキャッシュ: <https://joblib.readthedocs.io/en/latest/memory.html>
