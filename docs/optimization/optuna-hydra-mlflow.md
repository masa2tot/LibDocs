# Optuna × Hydra × MLflow の役割整理

Hydra をエントリポイントに据え、Optuna でハイパーパラメータ探索を行いながら MLflow へ結果を記録するワークフローは、研究・実務どちらでも使われる組み合わせです。ここでは、それぞれのライブラリが担う責務と、試行ごとに設定を調整する際の考え方を専門用語の解説付きでまとめます。

## Hydra と OmegaConf: 設定を安全にコピーする

- `@hydra.main` デコレータを付けた関数は、コマンドライン引数や設定ファイルを自動で読み込み、`DictConfig` として渡します。Hydra の基本的な使い方は [構成・制御 / Hydra](../config/hydra.md) で紹介しているとおりです。
- Hydra が返す `DictConfig` は原則として読み取り専用なので、試行ごとに値を書き換えるときは `OmegaConf.to_container(cfg, resolve=True)` で通常の辞書に変換し、`OmegaConf.create` で新しい設定オブジェクトを生成します。この手順は [OmegaConf 逆引き](../config/omegaconf.md) でも副作用を避ける方法として解説しています。
- GPU を切り替えたり補間を評価した状態で値を確定させたい場合も、コピーした設定に対して `tc.runtime.device` のように書き込むと安全です。元の `cfg` を直接変更すると、Hydra が保持する履歴と食い違う原因になります。

## Optuna: 試行 (Trial) と探索空間の定義

- Optuna の `Trial` オブジェクトは 1 回の評価を指し、`suggest_float` や `suggest_categorical` で探索空間を宣言します。試行番号 (`trial.number`) を活用すると、GPU の割り当てや出力先フォルダ名など、実行環境に応じた制御が可能です。
- `create_study` で生成した `Study` は全試行の履歴とベストスコアを保持し、`optimize` の `n_jobs` 引数により並列試行ができます。GPU 数が限られる場合は `torch.cuda.device_count()` で物理的な上限を調べ、必要に応じて `min(req_n_jobs, max(1, gpu_count))` のように計画的に調整します。
- 試行ごとの戻り値には評価指標（例: RMSE）のように最適化したいスカラー値を返します。Study の `best_params` や `best_value` は最後にまとめて参照できるので、結果をファイルへ保存したり外部へ通知するタイミングを自由に設計できます。

## MLflow: 実験ログの整理

- MLflow では Tracking URI と Experiment 名を先に設定しておくと、試行結果が正しいフォルダへまとまります。[MLflow Tracking を言葉から理解する](../tracking/mlflow-tracking.md) にある用語集を参考に、Run / Experiment / Artifact などの役割を押さえておきましょう。
- `mlflow.start_run` はコンテキストマネージャとして振る舞い、ブロック内で `log_params` や `log_metrics` を呼び出すと自動で Run に紐づきます。Optuna の試行を記録するときは Run 名に試行番号を含めると、どの組み合わせがどの結果を生んだか追跡しやすくなります。
- ハイパーパラメータ検索全体をまとめる親 Run を作り、その中で各試行をネストした Run として残す構成にすると、MLflow UI 上で探索全体の概要と個別試行を同時に振り返ることができます。成果物（例: JSON のサマリーや可視化画像）は `log_artifact` で保存し、後から再現性を担保できるようにしましょう。

## 実行リソースと成果物の後処理

- GPU が無い環境では CPU 実行へフォールバックし、`pin_memory` や `non_blocking` のような転送設定は False にするのが一般的です。GPU が存在するときのみ True に切り替えることで、不要な設定ミスを防げます。
- 探索結果をファイルにまとめる際は `Path` オブジェクトで出力先ディレクトリを日付ごとに作成し、Study のサマリー (`best_params`, `best_value`, `n_trials`) を JSON などの読みやすい形式で保存します。保存したファイルを MLflow のアーティファクトとしてアップロードすれば、Run の履歴と一緒に参照できます。

これらの役割分担を意識すると、「設定を用意する」「探索を回す」「結果を残す」という 3 つの作業が明確になり、複雑なハイパーパラメータ探索でも事故が起きにくくなります。
