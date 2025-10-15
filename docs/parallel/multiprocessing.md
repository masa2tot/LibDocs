# multiprocessing を基礎から理解する

`multiprocessing` は Python に最初から同梱されている標準ライブラリで、1 台のコンピュータの中で複数のプロセスを同時に動かすための
道具です。プロセスとは OS 上で動く独立したプログラムの単位のことで、互いにメモリを直接共有しないため、安全に同じ関数を同時実行
できます。ここでは専門用語を一つずつ説明しながら、仕組みと使い方を学びます。

## 用語の整理

| 用語 | 説明 |
| --- | --- |
| GIL (Global Interpreter Lock) | CPython が同時に 1 つのスレッドしか Python バイトコードを実行させない仕組み。CPU をたくさん使う処理ではボトルネックになる。 |
| CPU バウンド | CPU の計算時間が支配的な処理。行列計算や画像処理などが該当する。 |
| プロセスプール (`Pool`) | あらかじめ複数のプロセスを作っておき、順番に仕事を割り当てるための管理オブジェクト。 |
| キュー (`Queue`) | 先入れ先出し (FIFO) でデータをやり取りする仕組み。プロセス間で安全に情報を渡す。 |
| マネージャ (`Manager`) | 辞書やリストのような Python オブジェクトを共有し、複数プロセスが同じデータを扱えるようにする補助機能。 |

## インストール

追加のインストールは不要です。Python 3.8 以降が推奨されます。現在のバージョンは以下のコマンドで確認できます。

```bash
python --version
```

## 最初の例: 並列に関数を実行する

```python
from multiprocessing import Pool


def square(x: int) -> int:
    """引数の二乗を計算する単純な関数。"""
    return x * x


if __name__ == "__main__":  # Windows や macOS ではこのガードが必須
    with Pool(processes=4) as pool:  # 4 個のプロセスプールを作成
        results = pool.map(square, range(10))  # 0〜9 を順番に渡し、並列で計算
    print(results)
```

- `if __name__ == "__main__"` は Windows や macOS がプロセスを生成するときに、モジュール全体を再実行してしまう問題を避ける合言葉です。
- `Pool(processes=4)` は同時に最大 4 個のプロセスが走ることを意味します。`os.cpu_count()` で自動的に CPU 数を調べるのも有効です。
- `pool.map` は組み込み関数 `map` と同じ API で使え、複数の引数を順番に処理します。

## 代表的な API とその意味

| API | 何ができるか | いつ使うか |
| --- | --- | --- |
| `multiprocessing.Process` | 任意の関数を新しいプロセスとして実行する。 | 細かい制御をしたいとき。プロセスを自分で開始・終了したい場合。 |
| `multiprocessing.Pool` | プロセスプールを使って関数を繰り返し呼び出す。 | 同じ関数を大量の入力に適用したいとき。 |
| `multiprocessing.Queue` | プロセス間で順序付きのメッセージを渡す。 | ワーカーにタスクを配ったり、結果を受け取ったりするとき。 |
| `multiprocessing.Manager` | 共有辞書やリスト、ロックなどを提供する。 | プロセス間で状態を共有したいとき。 |
| `multiprocessing.shared_memory` | NumPy 配列のような大きなデータをコピーせず共有する。 | 大きな配列を頻繁に読み書きしたいとき。 |

## ベストプラクティス

1. **プロセス数は計画的に** – CPU バウンドな処理では `os.cpu_count()` と同じ数まで増やすのが目安です。ただし I/O 待ちが多い場合は
   少し多めにしても問題ありません。
2. **共有リソースを保護する** – 同じファイルやデータベースに同時アクセスするときは、`Lock` や `Semaphore` を使って順番を守ります。
3. **例外を見逃さない** – `Pool.apply_async(..., error_callback=...)` を使うと、子プロセスで起きたエラーをメインプロセスで拾えます。
4. **終了処理を丁寧に** – `pool.close()` で新しいジョブの受付を止め、`pool.join()` で完了を待つ習慣をつけましょう。`with` 文を使うと自
   動で呼び出されるので安全です。
5. **デバッグはシリアルで確認** – 複数プロセスでのトラブルは切り分けが難しいため、まずは通常の関数呼び出しで動くかを確かめてから並列化
   するのが近道です。

## もっと学ぶには

- 公式ドキュメント: <https://docs.python.org/3/library/multiprocessing.html>
- サンプルコード集: <https://github.com/python/cpython/tree/main/Lib/multiprocessing>
- プログラミングガイドライン: <https://docs.python.org/3/library/multiprocessing.html#programming-guidelines>
