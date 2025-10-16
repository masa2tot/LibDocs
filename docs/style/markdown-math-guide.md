# Markdownで数式を書くときのベストプラクティス

LaTeX 非対応の環境でも崩れず、LaTeX 対応のビューアでは正しく描画されるようにするための書き方ガイドです。ライブラリの利用解説を書く際に数式が必要になったら、このページを参照してください。

## 基本方針

1. **すべての数式は `$...$` または `$$...$$` で囲む**  
   インラインは `$...$`、ブロックは `$$...$$` を使います。
2. **ベクトル・行列は `\boldsymbol{}` を使う**  
   例: `\boldsymbol{v}` → $\boldsymbol{v}$。`\vec{}` は環境によって文字化けするため避けます。
3. **添字や上付きには必ず `{}` を付ける**  
   例: `$v_x$`, `$x_i$`, `$A_{ij}$`, `$x^{(t)}$`。
4. **環境依存の文字を避ける**  
   全角スペースや全角記号（例: ＋, －）ではなく、半角記号（`+`, `-`, `*`, `/`, `=`）を用います。
5. **文中の変数も `$...$` で囲む**  
   例: `$p$ は圧力、$\rho$ は密度、$\boldsymbol{v}$ は速度ベクトルを表す。`

## インライン数式の例

```markdown
流体の速度ベクトルを $\boldsymbol{v}$、密度を $\rho$ とすると、連続の式は $\frac{\partial \rho}{\partial t} + \nabla \cdot (\rho \boldsymbol{v}) = 0$ と表される。
```

表示例：

流体の速度ベクトルを $\boldsymbol{v}$、密度を $\rho$ とすると、連続の式は $\frac{\partial \rho}{\partial t} + \nabla \cdot (\rho \boldsymbol{v}) = 0$ と表される。

## ブロック数式の例

```markdown
$$
\rho \frac{D \boldsymbol{v}}{D t}
= -\nabla p + \mu \nabla^2 \boldsymbol{v} + \rho \boldsymbol{g}
$$
```

表示例：

$$
\rho \frac{D \boldsymbol{v}}{D t}
= -\nabla p + \mu \nabla^2 \boldsymbol{v} + \rho \boldsymbol{g}
$$

## 記号・演算子の推奨書き方

| 用途 | 推奨書き方 | 表示 | 備考 |
| --- | --- | --- | --- |
| ベクトル | `\boldsymbol{v}` | $\boldsymbol{v}$ | 一貫して使用 |
| 勾配 | `\nabla f` | $\nabla f$ |  |
| 発散 | `\nabla \cdot \boldsymbol{v}` | $\nabla \cdot \boldsymbol{v}$ |  |
| 回転（渦度） | `\nabla \times \boldsymbol{v}` | $\nabla \times \boldsymbol{v}$ |  |
| 実質微分 | `\frac{D}{D t}` | $\frac{D}{D t}$ |  |
| 偏微分 | `\frac{\partial f}{\partial x}` | $\frac{\partial f}{\partial x}$ |  |
| 絶対値 | `|\boldsymbol{v}|` | $|\boldsymbol{v}|$ |  |
| 添字付き | `x_i`, `A_{ij}` | $x_i$, $A_{ij}$ |  |
| 上付き | `x^{(t)}` | $x^{(t)}$ | 時刻・次数などに使用 |

## 実践的なヒント

- 長い式では `align` や `split` などの環境は避け、1 行に収まるよう簡潔にします。Markdown では改行位置によって崩れる場合があります。
- コメントや注釈内の `$...$` にはスペースを入れないでください。例: `$\rho$` と書き、`$ \rho $` は避けます。
- 記号の意味は本文で必ず説明して可読性を保ちます。例: `$p$（圧力）`、`$\boldsymbol{g}$（重力加速度ベクトル）`。

## 推奨フォーマットのサンプル

```markdown
### 方程式タイトル

文中で各変数を `$...$` で囲み、`\boldsymbol{}` を使ってベクトルを明示します。

$$
（ここに数式）
$$

ここで、
- $\rho$: 密度
- $\boldsymbol{v}$: 速度ベクトル
- $p$: 圧力
- $\boldsymbol{g}$: 重力加速度ベクトル
```

## まとめ

| 原則 | 内容 |
| --- | --- |
| **インライン数式** | `$...$` で囲む |
| **ブロック数式** | `$$...$$` で囲む |
| **ベクトル** | `\boldsymbol{}` を使う |
| **添字・上付き** | `{}` を忘れずに |
| **環境非依存** | 全角文字や `\vec` を避ける |
| **説明の一貫性** | 文中でも `$...$` を使う |
