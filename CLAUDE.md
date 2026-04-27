# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## このフォルダの性格

コードを書く場所ではなく、**画像生成 MCP / Skill 一式の運用ハブ**。新しい画像系ツールの導入・更新・動作検証はこのフォルダで Claude Code セッションを開いて行う。コミットされる「アプリケーションコード」は無い。

ユーザー向けの導入手順・トラブルシューティングは `README.md` に集約済み。本ファイルでは Claude が知っておくべき**運用上の制約と非自明な構成**だけ補足する。

## 構成要素と接続関係

| 要素 | 実態 | 起動 |
|------|------|------|
| `openai-image` MCP | `kazyam53/openai_gen_image_mcp`、Python 製。実体 `C:\Users\kite_\.local\bin\openai-gen-image-mcp.exe` | `~/.claude.json` の `mcpServers.openai-image.env.OPENAI_API_KEY` に**平文で**埋め込まれている |
| `excalidraw` MCP | `mcp_excalidraw/dist/index.js`（入れ子のローカルクローン、`.gitignore` 済み） | MCP 本体は Claude Code 起動時に自動起動。**描画用の HTTP サーバは別途起動が必要**（後述） |
| `claude-mermaid` | npm global、MCP サーバ専用。CLI 単体では描画不可 | MCP として登録するには `claude mcp add --scope user mermaid claude-mermaid` |
| Anthropic Skills | `example-skills@anthropic-agent-skills` プラグイン経由で一括導入。`canvas-design` / `algorithmic-art` / `slack-gif-creator` ほか | `Skill` ツールで起動 |

## Excalidraw を使うときの手順

`mcp__excalidraw__create_from_mermaid` や `export_to_image` は**ブラウザ側の DOM が必要**。MCP の接続が `✓ Connected` でも、以下を満たさないと無音で失敗する:

1. `mcp_excalidraw/` で `npm run canvas` を起動（port 3000、バックグラウンド可）
2. ブラウザで `http://127.0.0.1:3000/` を開いておく（`start http://127.0.0.1:3000/`）
3. それから `mcp__excalidraw__*` を呼ぶ

`batch_create_elements` だけはサーバ側ステートで完結するのでブラウザ無しで動く。`describe_scene` はそのサーバ側ステートを返す。

**Path traversal 制約**: `export_to_image` の `filePath` は CWD（このフォルダ）配下しか許可されない。`Pictures/claude-generated/` のような外部パスを直接指定するとエラー。一旦ローカルに書いて `mv` で移動するか、`EXCALIDRAW_EXPORT_DIR` 環境変数で許可ベースを変更する。

## ⚠️ 上流パッケージへの手書きパッチ（再インストールで消える、再適用必須）

このフォルダを運用するうえで、**外部 npm/uv パッケージの中身を直接書き換えている箇所が複数ある**。本体を更新（`npm i -g`、`uv tool upgrade` 等）するとパッチは消えるので、ここを**チェックリストとして必ず参照**すること。

### パッチ適用箇所一覧

| # | ファイル | 目的 | 復元コマンドで消えるトリガー |
|---|---------|------|------------------------------|
| 1 | `C:\Users\kite_\AppData\Roaming\npm\node_modules\claude-mermaid\build\handlers.js` 41 行目付近 | `execFileAsync("npx", args)` → `process.platform === "win32"` 分岐で `cmd.exe /c npx` | `npm i -g claude-mermaid` |
| 2 | 同ファイル 64 行目付近 | ブラウザ起動 `spawn("start", ...)` → `cmd.exe /c start "" <url>`、`child.on("error", ...)` ハンドラ追加 | 同上 |
| 3 | `C:\Users\kite_\AppData\Roaming\npm\node_modules\claude-mermaid\build\serve.js` 44 行目付近 | ギャラリー起動を `cmd.exe /c start ""` 経由に | 同上 |
| 4 | 同 `build/index.js` の `TOOL_DEFINITIONS` 内 `mermaid_preview` の `description` | Spotter フレンドリ化（USE WHEN / DO NOT USE WHEN / 代替ツール明示） | 同上 |
| 5 | 同 `TOOL_DEFINITIONS` 内 `mermaid_save` の `description` | USE WHEN / PRECONDITION 明示 | 同上 |
| 6 | `C:\Users\kite_\AppData\Roaming\uv\tools\openai-gen-image-mcp\Lib\site-packages\src\server.py` の `generate_image` docstring | Spotter フレンドリ化（USE WHEN / DO NOT USE WHEN / PRECONDITIONS / OUTPUT） | `uv tool upgrade openai-gen-image-mcp` または再インストール |
| 7 | 同ファイルの `edit_image` docstring | 同上 | 同上 |

### 再適用が必要な兆候

- `mermaid_preview` で `Error rendering Mermaid diagram: spawn npx ENOENT` が出たら → #1〜#3 が剥がれている
- ツール説明から「USE WHEN」「DO NOT USE WHEN」のセクションが消えていたら → #4〜#7 が剥がれている

### 罠の正体

`spawn`/`execFile` 系の問題（#1〜#3）は `caveat` に記録済み。`caveat_search "windows npx spawn"` で `windows-claude-mermaid-mcp-fails-with-spawn-npx-enoent-...` がヒットする。根本治療は上流（`claude-mermaid` 作者）への PR/Issue。

### Spotter 側の既知問題（解決済み・参考）

過去、`spotter db refresh` / `rebuild` 実行時に `mermaid` / `openai-image` の自動収集が失敗していた:

- `mermaid`: Spotter 自身が `claude-mermaid`（Windows の `.cmd` ラッパー）を素の `spawn` で起動して ENOENT
- `openai-image`: Spotter が MCP を起動するとき Claude Code 設定の env（`OPENAI_API_KEY`）を引き継がず起動直後に失敗

→ **claude-spotter 1.2.2 で両方解決**。`spotter db rebuild` で 4 ツール（`mcp__mermaid__mermaid_preview` / `mcp__mermaid__mermaid_save` / `mcp__openai-image__generate_image` / `mcp__openai-image__edit_image`）が自動収集されるので、tool-db.json への手書き投入は不要。1.2.2 未満に戻すと再発する可能性あり。



## OpenAI 課金まわりの落とし穴（実害あり）

- **ChatGPT Plus と従量課金は完全に別の財布**。Plus に入っていても画像生成 API は使えない。
- `gpt-image-2` は **OpenAI 組織の本人確認が必須**。未確認だと `Your organization must be verified to use the model 'gpt-image-2'` で 403。確認は https://platform.openai.com/settings/organization/general、反映に最大 15 分。
- 残高ゼロで叩くと `billing_hard_limit_reached` で 400。https://platform.openai.com/settings/organization/billing/overview を最初に確認。
- API キーが MCP 設定に平文で入っている件は**ユーザー合意済み**（Windows ユーザー環境変数経由が起動済みプロセスに伝播しないため）。改善余地として記録だけ残してある。

## 慣習

- 生成画像の出力先は `C:\Users\kite_\Pictures\claude-generated\`。新ツール追加時もここに揃える。
- `.env` は git ignored。`.env.example` がテンプレ。
- `mcp_excalidraw/` 配下は upstream の別リポジトリ。**この配下を編集してはいけない**（差分が pull で消える）。設定変更は親フォルダの `~/.claude.json` 側で行う。

## 動作確認の流れ

`README.md` の「インストール後の検証」節が一次資料。新しいツールを足したら同じスタイルで章を追加する。
