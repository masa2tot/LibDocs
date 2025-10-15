# Joblib で並列処理とキャッシュを手軽に

Joblib は、複雑な設定なしで並列処理と結果の再利用（キャッシュ）を行える Python ライブラリです。`multiprocessing` を直接扱うより
も短いコードで書けるように設計されており、scikit-learn などの機械学習ライブラリでも広く使われています。ここではキーワードを丁寧
に説明しながら、Joblib の基本を押さえます。

## 用語の整理

| 用語 | 説明 |
| --- | --- |
| バックエンド | 並列化の実装方式を切り替える仕組み。`processes`（プロセス方式）や `threading`（スレッド方式）などが選べる。 |
| シリアライズ | オブジェクトをファイルやメモリに保存できる形式へ変換すること。Joblib には NumPy 配列向けの高速シリアライザがある。 |
| `Parallel` クラス | 並列実行の司令塔。いくつジョブを投げるか、どのバックエンドを使うかを決める。 |
| `delayed` 関数 | 関数と引数をセットにして「後で実行する仕事」としてキューに入れるラッパー。 |
| キャッシュ (`Memory`) | 計算結果をディスクに保存し、同じ入力で呼び出したときに再計算を省く仕組み。 |

## インストール

```bash
pip install joblib
```

- Python 3.8 以上が推奨です。
- scikit-learn と組み合わせるときは `pip install scikit-learn` も同時に行うと便利です。

## 最初の例: 文章の前処理を並列化する

```python
from joblib import Parallel, delayed


def preprocess(sample: str) -> str:
    """文字列を小文字に変換するシンプルな処理。"""
    return sample.lower()


corpus = ["Data", "Science", "Pipeline"]
results = Parallel(n_jobs=2)(delayed(preprocess)(text) for text in corpus)
print(results)
```

- `Parallel(n_jobs=2)` は同時に 2 個までジョブ（個々の処理単位）を走らせる設定です。`n_jobs=-1` とすると利用可能な CPU をすべて使います。
- `delayed(preprocess)(text)` の形で書くと、引数を保持した「まだ実行していない関数」を作れます。`Parallel(...)` がそれらを順番に取り出し
  て実行します。

## 代表的な機能と使いどころ

| 機能 | 何ができるか | いつ使うか |
| --- | --- | --- |
| `Parallel` | 並列マップ処理、進捗ログの出力 (`verbose`)、例外のまとめ取りなどを提供。 | 同じ関数を大量のデータに適用したいとき。 |
| `delayed` | 引数付きの関数呼び出しを「実行予約」に変換。 | for ループで複数パラメータを組み合わせたいとき。 |
| `Memory` | 結果をディスクにキャッシュし、同じ入力での再実行を省略。 | 前処理や特徴量計算の時間を短縮したいとき。 |
| `parallel_backend` | `loky`（プロセスベース）、`threading`（スレッドベース）などをコンテキストマネージャで切り替える。 | I/O が多いか計算が多いかで適切な実行方式を選びたいとき。 |
| `register_parallel_backend` | 自作のバックエンドを登録。 | 特殊なクラスタ環境で Joblib を使いたいとき。 |

## キャッシュの例

```python
from joblib import Memory

memory = Memory(location="./cache", verbose=0)


@memory.cache
def heavy(x: int) -> int:
    print("計算中...")
    return x * x


print(heavy(10))  # 最初は関数が実際に実行される
print(heavy(10))  # 2 回目以降はディスクに保存された結果を返す
```

- `Memory(location="./cache")` はカレントディレクトリに `cache` フォルダを作り、結果をファイルとして保存します。
- `@memory.cache` デコレータ（関数に追加情報を与える記法）を使うと、自動的にキャッシュが有効になります。

## ベストプラクティス

1. **バックエンドを選ぶ基準を知る** – 計算量が多い CPU バウンドな処理ではデフォルトの `loky`（プロセスベース）が有利です。待ち時間が長い
   I/O バウンドな処理では `threading` を選ぶとプロセス生成コストを抑えられます。
2. **エラー情報を集める** – `Parallel(..., verbose=10)` などログ出力を増やすと、どのジョブで失敗したかが分かりやすくなります。`batch_size`
   や `prefer` 引数も記録しておくと再現が容易です。
3. **キャッシュの失効条件を決める** – 入力データの更新やコード変更があったら `memory.clear()` でキャッシュを掃除します。古い結果を参照しない
   ように注意しましょう。
4. **scikit-learn との連携を理解する** – 多くの scikit-learn モデルは `n_jobs` 引数を持っており、内部で Joblib を呼び出します。同じマシン
   で二重に並列化しないよう、外側と内側の `n_jobs` を調整しましょう。
5. **テスト時はシリアル実行に切り替える** – 並列化が不要なユニットテストでは `with parallel_backend("sequential"):` と書くことで 1 ジョブだけを
   順番に実行できます。テストの安定性が上がります。

## 参考リンク

- 公式ドキュメント: <https://joblib.readthedocs.io/>
- 並列計算ガイド: <https://joblib.readthedocs.io/en/latest/parallel.html>
- キャッシュの詳細: <https://joblib.readthedocs.io/en/latest/memory.html>
