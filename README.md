# claude-image-tools

Claude Code から画像生成・処理を呼ぶための MCP / Skill 一式の管理フォルダ。

## このフォルダの役割

- 画像系ツールの導入スクリプト・設定・運用メモを一箇所にまとめる
- Excalidraw MCP のような git clone するソースをここに置く（`.gitignore` で本体は除外）
- 新規導入・更新作業はこのフォルダで Claude Code セッションを開いてやる
- auto memory を「画像ツール」用の project に紐付ける

## 入っているもの（インストール後）

| 名前 | 種別 | 用途 | API key |
|------|------|------|---------|
| openai-image | MCP | GPT Image 2.0 でラスター生成（OG banner / README hero） | OpenAI |
| excalidraw | MCP | 手書き風ダイアグラム | 不要 |
| claude-mermaid | npm global | Mermaid 図のライブプレビュー | 不要 |
| Canvas Design | Anthropic Skill | museum 品質 PNG/PDF | 不要 |
| Algorithmic Art | Anthropic Skill | p5.js 生成アート | 不要 |
| Slack GIF Creator | Anthropic Skill | アニメ GIF | 不要 |

## 導入手順

このフォルダで Claude Code を起動し、最初に以下を投げる:

```
このフォルダの plan を実行して: C:\Users\<your-username>\.claude\plans\gpt-image-2-0-woolly-scott.md
```

セッションが Step 1 以降を順に走らせる。

## API key

`.env` に置く（git ignore 済み）。例は `.env.example` 参照。

## 出力先

- 生成画像: `C:\Users\<your-username>\Pictures\claude-generated\`

## インストール後の検証（再起動後セッションで実行）

API キーを Windows ユーザー環境変数に設定したあと、Claude Code を再起動してから以下を順に実行する。

### 1. MCP 接続確認

```bash
claude mcp list
```

期待: 以下が `✓ Connected` で並ぶ
- `excalidraw`
- `openai-image`

### 2. GPT Image 2.0 で英文画像

新セッションで:
> openai-image MCP で 1024x1024 の画像を生成して。プロンプトは "a small cute cat sitting on a wooden table, soft lighting"。output_path は `C:/Users/<your-username>/Pictures/claude-generated/test-cat.png`

### 3. 日本語テキスト埋め込み（GPT Image 2.0 の主要価値）

> openai-image MCP で 1200x630 の画像を作って。中央に大きく "Caveat" の文字、下に小さく "罠を記録する OSS" の日本語、背景は深い青のグラデーション。output_path は `C:/Users/<your-username>/Pictures/claude-generated/test-og-banner.png`

文字が崩れずに描画されることを確認。

### 4. Excalidraw / Mermaid / Canvas Design

- > Excalidraw MCP で 3 ノードのアーキ図を描いて
- > claude-mermaid でフローチャートを作って: A → B → C
- > Canvas Design skill を使って Caveat 用 OG banner 案を作って

### トラブルシューティング

- `openai-image` が `Failed to connect` の場合:
  - PowerShell で `[Environment]::GetEnvironmentVariable("OPENAI_API_KEY", "User")` を実行して空でないことを確認
  - 空なら再設定 → Claude Code を完全終了して再起動
- OpenAI API rate limit エラー: billing コンソールで tier 確認、必要なら課金枠を上げる

## インストール内訳（実装ノート）

| 項目 | 実態 |
|------|------|
| openai-image MCP | `kazyam53/openai_gen_image_mcp` を `uv tool install` 経由で導入。実体は `C:\Users\<your-username>\.local\bin\openai-gen-image-mcp.exe`。Python 製、`gpt-image-2` がデフォルト |
| excalidraw MCP | `yctimlin/mcp_excalidraw` をローカル clone & build。実体は `mcp_excalidraw/dist/index.js` |
| claude-mermaid | npm global 1.6.2 |
| Anthropic Skills | `example-skills@anthropic-agent-skills` 一括インストール（canvas-design / algorithmic-art / slack-gif-creator / brand-guidelines / frontend-design / mcp-builder / theme-factory ほか同梱） |

## 将来の追加候補

- Recraft V4 — SVG ロゴ用（必要時）
- fal.ai — 背景除去・アップスケール・インペイント（後処理が要る時）
- Nano Banana 2/Pro — Gemini 系。GPT Image 2.0 でカバーできない用途が出た時に
