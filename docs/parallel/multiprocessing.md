# multiprocessing

Python 標準ライブラリに含まれる `multiprocessing` は、プロセスベースの並列実行を提供し、GIL を回避しながら CPU バウンドなタスクを加速させます。公式ドキュメントの構成に従い、ここでは基礎概念から高度な利用までを整理します。

## 概要

- **主な用途**: CPU 集約型タスクの並列化、長時間実行タスクの分散
- **実装モデル**: プロセス間通信 (IPC) を用いたプリフォークモデル
- **強み**: 標準ライブラリで依存が不要、`Process`/`Pool` を中心に豊富なプリミティブを提供

## インストール

追加インストールは不要です。Python 3.8 以降を推奨します。

```bash
python --version  # 3.11 など
```

## クイックスタート

```python
from multiprocessing import Pool


def square(x: int) -> int:
    return x * x

if __name__ == "__main__":
    with Pool(processes=4) as pool:
        results = pool.map(square, range(10))
        print(results)
```

- `if __name__ == "__main__"` ガードは Windows / macOS で必須です
- `Pool` はプロセスプールを生成し、関数を並列に適用します

## API ハイライト

| API | 説明 |
| --- | --- |
| `multiprocessing.Process` | 任意のターゲット関数を独立プロセスで実行 |
| `multiprocessing.Pool` | プロセスプールによる並列マップ・スターマップ |
| `multiprocessing.Queue` | プロセス間の FIFO 通信 |
| `multiprocessing.Manager` | 共有オブジェクト（dict/list/Namespace 等）の提供 |
| `multiprocessing.shared_memory` | Numpy 配列などの高効率な共有メモリ |

## ベストプラクティス

1. **明示的なエントリーポイント**: `spawn` モードのプラットフォームではモジュールレベルの副作用を避ける
2. **プロセス数の制御**: `os.cpu_count()` を上限とし、I/O 待ちが多い場合は余裕をもたせる
3. **共有資源の同期**: `Lock` や `Semaphore` を活用しデッドロックを防止
4. **エラーハンドリング**: `Pool.apply_async(..., error_callback=...)` で例外を収集
5. **クリーンアップ**: `Pool.close()` → `Pool.join()` の順で終了を待機

## 関連リファレンス

- 公式ドキュメント: <https://docs.python.org/3/library/multiprocessing.html>
- サンプル集: <https://github.com/python/cpython/tree/main/Lib/multiprocessing>
- ベストプラクティス: <https://docs.python.org/3/library/multiprocessing.html#programming-guidelines>
