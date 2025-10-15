# uv 入門

uv は Astral が開発する高速な Python パッケージマネージャーです。`pip` と `pip-tools` を統合したような操作体系で、依存関係の解決からインストールまでを一貫して行えます。Rust 製のためインストールが非常に速く、大規模プロジェクトでも CI の待ち時間を短縮できます。

## 概要

| 用語 | 意味 | メモ |
| --- | --- | --- |
| `uv pip compile` | `requirements.in` や `pyproject.toml` から依存関係を解決し、ロックファイル（`uv.lock` など）を生成するコマンド。 | `pip-compile` と同等の機能を持つが、解決速度が速い。 |
| `uv sync` | ロックファイルに基づき、仮想環境へ必要なパッケージをまとめてインストールする。 | venv のパスを指定でき、`pip install -r` より再現性が高い。 |
| `uv run` | 一時的な環境でコマンドを実行する。 | `npx` のように、ローカルに未インストールのツールをすぐに実行できる。 |

## インストール

シングルバイナリなので、curl かパッケージマネージャーから簡単に導入できます。

```bash
# Linux / macOS (x86_64)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows (PowerShell)
irm https://astral.sh/uv/install.ps1 | iex
```

インストール後は `~/.local/bin` など PATH に追加されたディレクトリに `uv` バイナリが配置されます。`uv --version` で正しく導入されたか確認しましょう。

## クイックスタート

1. 依存関係を宣言します。`requirements.in` でも `pyproject.toml` でも構いません。
2. `uv pip compile pyproject.toml` を実行し、`uv.lock` を生成します。
3. 仮想環境にインストールする場合は `uv sync --python .venv/bin/python` を実行します。

```bash
# プロジェクト初期化例
uv init my-app
cd my-app
uv add hydra-core mlflow
uv pip compile pyproject.toml
python -m venv .venv
uv sync --python .venv/bin/python
```

`uv init` コマンドは `pyproject.toml` や `src/` ディレクトリを含む最小構成を自動生成してくれるため、研究用リポジトリを素早く立ち上げたいときに便利です。

## よく使うオプション

- `--python VERSION` — プロジェクトが想定する Python バージョンを明示します。CI での互換性確認に有効です。
- `--preview` — 未安定な依存解決アルゴリズムを試すときに指定します。通常は不要です。
- `uv export --format requirements.txt` — CI で `pip install -r` を使う必要がある場合に変換できます。

## ベストプラクティス

- `uv.lock` は必ずバージョン管理し、レビューで差分を確認できるようにします。
- 依存関係を追加・更新するときは `uv add` / `uv pip compile` をセットで実行し、ロックファイルを最新化します。
- CI では `uv sync --frozen` を使い、ローカルと同じバージョンのみインストールされるようにしましょう。

## 関連リファレンス

- [公式 CLI リファレンス](https://docs.astral.sh/uv/reference/cli/)
- [uv と pip の比較記事（英語）](https://docs.astral.sh/uv/concepts/compare/)
