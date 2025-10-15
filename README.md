# LibDocs

LibDocs は、機械学習研究で頻用される公式ライブラリの実践的な使い方をまとめた MkDocs ベースのドキュメントサイトです。レイヤ構造と並列実行の観点から技術を体系化し、公式ドキュメントに準じた解説を提供します。公開サイトは <https://masa2tot.github.io/LibDocs/> から閲覧できます。

## ディレクトリ構成

```
├── mkdocs.yml          # サイト設定
└── docs/
    ├── index.md        # サイトトップ。レイヤ構成と編集ガイド
    └── parallel/       # 並列実行ライブラリの公式風ドキュメント
        ├── index.md
        ├── multiprocessing.md
        ├── joblib.md
        ├── ray.md
        └── hydra-launcher.md
```

- 今後はレイヤごとにサブディレクトリを追加し、各ライブラリをページとして整理します
- すべてのページは「概要 / インストール / クイックスタート / API ハイライト / ベストプラクティス / 関連リファレンス」で統一

## 並列実行セクション

`docs/parallel/` では以下のライブラリを扱います。

| ライブラリ | ハイライト |
| ---------- | ---------- |
| `multiprocessing` | Python 標準のプロセス並列。GIL を回避して CPU バウンド処理を加速 |
| `joblib` | `Parallel`/`delayed` によるシンプルな並列マップと結果キャッシュ |
| `Ray` | 分散タスク、Tune による大規模ハイパーパラメータ探索 |
| `Hydra Launchers` | Hydra の `multirun` をバックエンドに合わせて並列化 |

## ローカルでのプレビュー

```bash
pip install -r requirements.txt
mkdocs serve
```

ブラウザで <http://127.0.0.1:8000/> を開くとライブリロード付きで確認できます。

## 公開と自動デプロイ

- GitHub Pages での公開 URL: <https://masa2tot.github.io/LibDocs/>
- GitHub Actions のワークフロー (`.github/workflows/deploy-mkdocs.yml`) が `main` ブランチへのプッシュをトリガーにサイトをビルドし、`gh-pages` ブランチ経由で公開します。
- GitHub 側では **Settings → Pages** で "Source" を "GitHub Actions" に設定してください。

ワークフローをローカルで検証するには、次のコマンドで静的サイトをビルドします。

```bash
pip install -r requirements.txt
mkdocs build --strict
```

## 今後の拡張

- 設定管理（Hydra / OmegaConf）や追跡（MLflow）など、他レイヤのセクションを順次追加
- 各ライブラリごとのサンプルコードや実験テンプレートを `examples/` として公開予定
- プレビュー用ブランチやバージョン別公開など、GitHub Pages を活用した運用フローを強化
