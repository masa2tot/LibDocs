# venv の基礎

`venv` は Python 標準ライブラリが提供する仮想環境ツールです。プロジェクトごとに独立したパッケージインストール領域を作り、システムの Python や他プロジェクトに影響を与えずに依存関係を管理できます。

## 概要

| 用語 | 意味 | メモ |
| --- | --- | --- |
| 仮想環境 (Virtual Environment) | Python 実行ファイルと標準ライブラリをコピーし、独自の site-packages を持つ隔離環境。 | `.venv/` や `env/` などプロジェクトルートに作成するのが一般的。 |
| アクティベート | 仮想環境を利用するために PATH を書き換える操作。 | シェルごとにコマンドが異なるので注意。 |
| `pyproject.toml` | Python プロジェクトのメタデータや依存関係を宣言する標準ファイル。 | uv や Poetry など他ツールと共存可能。 |

## 仮想環境の作成手順

```bash
# Python 3.11 を利用する例
python3.11 -m venv .venv

# Linux / macOS の場合
source .venv/bin/activate

# Windows (PowerShell) の場合
.venv\\Scripts\\Activate.ps1

# 依存関係のインストール
pip install --upgrade pip
pip install -r requirements.txt
```

`activate` コマンドでシェルの先頭に `(.venv)` と表示されれば仮想環境が有効化されています。作業が終わったら `deactivate` で元に戻すことができます。

## uv との連携

uv を併用すると、以下の手順で再現性が高まります。

1. `python -m venv .venv` で環境を準備。
2. `uv pip compile pyproject.toml` でロックファイルを生成。
3. `uv sync --python .venv/bin/python` で `.venv` に依存関係を展開。

`uv sync` は自動的に `pip` を最新化し、インストール済みパッケージとの差分だけを更新してくれるため、`pip install -r` より高速です。

## 環境の共有と自動化

- **構成ファイル**: `.python-version` (pyenv 用) や `.venv` シンボリックリンクを用意しておくと、同僚が迷いません。
- **Makefile / タスクランナー**: `make init` といったタスクに `python -m venv` や `uv sync` をまとめると、コマンドを覚える負担が減ります。
- **CI での利用**: `python -m venv` は OS に依存しないため、GitHub Actions などでも同じ手順を再利用できます。

## トラブルシューティング

| 症状 | 原因 | 対処 |
| --- | --- | --- |
| `command not found: python` | システムに `python` コマンドが存在しない。 | `python3` や明示的なバージョンを指定して実行する。 |
| 仮想環境有効化後も `which python` がシステムを指す | `source` を忘れている、あるいは `.venv` のパスが異なる。 | `source .venv/bin/activate` を再実行し、`echo $VIRTUAL_ENV` を確認する。 |
| Windows でスクリプトが実行できない | 実行ポリシー (ExecutionPolicy) の制限。 | PowerShell を管理者権限で開き `Set-ExecutionPolicy RemoteSigned` を実行。 |

## ベストプラクティス

- 仮想環境ディレクトリ名は `.venv` に統一すると IDE の自動検出が動きやすくなります。
- プロジェクトルートに `README` や `docs/` で環境構築手順を明示し、コマンドをコピペできるようにします。
- 不要になった仮想環境は `rm -rf .venv` で削除し、再度 `python -m venv` から作り直す方がトラブルを減らせます。

## 関連リファレンス

- [Python 公式ドキュメント: venv](https://docs.python.org/ja/3/library/venv.html)
- [pyenv + venv の連携例](https://github.com/pyenv/pyenv)
