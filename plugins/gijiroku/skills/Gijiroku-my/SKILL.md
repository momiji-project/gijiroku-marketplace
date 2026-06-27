---
name: Gijiroku-my
description: 自分の最新 Lark Minutes（妙記）から、要約・タスク・図解入りの議事録ドキュメントを自動作成する。初回はチャット上のウィザードで保存先フォルダと認証を設定する。ユーザーが /Gijiroku-my と入力したとき、または「私の最新の録画から議事録を作って」と言ったときに使用する。
---

# Gijiroku-my — 自分の最新 Minutes から議事録を作成（配布版）

このスキルは **Lark（飞书 / Lark）の録画（Minutes / 妙記）** から議事録ドキュメントを自動生成します。
テナント固有の値（保存フォルダ等）は**コードに埋め込まず、ローカル設定ファイルから読み込みます**。

**CRITICAL — 実行前に必ず以下を Read ツールで読むこと（順番通り）。これらは `lark-cli` を導入すると同梱されます：**
1. `~/.claude/skills/lark-shared/SKILL.md` — 認証・身份・権限処理（**特に Split-Flow 認証 / QR 生成 / exit 10 の扱い**）
2. `~/.claude/skills/lark-minutes/SKILL.md` — Minutes 操作
3. `~/.claude/skills/lark-vc/references/vc-domain-boundaries.md` — 会議産物の領域境界
4. `~/.claude/skills/lark-doc/SKILL.md` — ドキュメント作成
5. `~/.claude/skills/lark-doc/references/lark-doc-xml.md` — XML 構文ルール
6. `~/.claude/skills/lark-doc/references/style/lark-doc-create-workflow.md` — 作成ワークフロー

> もし上記が見つからない場合は `lark-cli` が未導入。**Step 0** の手順で導入する。

---

## ローカル設定ファイル

| 項目 | 値 |
|------|-----|
| 設定ファイルのパス | `~/.claude/gijiroku-config.json`（`$HOME` を展開した絶対パスで読む） |

設定ファイルの形（初回ウィザードが書き込む。**このスキルにトークンをハードコードしない**）：

```json
{
  "folder_token": "",
  "tag": "(議事録)",
  "diagram_ref_doc": ""
}
```

- `folder_token`：議事録を保存する Lark ドライブのフォルダ token（フォルダ URL から自動抽出）
- `tag`：ファイル名に付ける目印（任意。空でも可）
- `diagram_ref_doc`：図解スタイル参照ドキュメントの token（任意。空なら本スキル末尾の「図解カタログ」を使う）

---

## Step 0: 初回セットアップ・ウィザード（plan モード的に1問ずつ確認）

> 目的：**ターミナル操作が苦手な相手でも、チャットで質問に答えるだけ**で使えるようにする。
> コマンドはすべて AI（あなた）が代行し、実行前に何をするか平易な言葉で一言伝えること。

### 0-1. 設定ファイルの確認

`~/.claude/gijiroku-config.json` を Read する。
- 読めて `folder_token` が**空でない** → セットアップ済み。**0-2 の認証チェックのみ**行い、OK なら Step 1 へ。
- 無い／壊れている／`folder_token` が空 → 0-2 以降のセットアップを実施。

### 0-2. lark-cli の確認

`lark-cli --version` を実行。
- 見つからない場合：ユーザーに「議事録機能には `lark-cli` の導入が必要です」と伝え、了承を得てから次を実行（Node.js/npm が前提）：
  ```bash
  npm install -g @larksuite/cli
  ```
  - `npm` 自体が無ければ、Node.js（npm 同梱）の導入を案内してから再実行。
- 導入後は **CRITICAL の参照ファイル**が `~/.claude/skills/` 配下に揃う。揃っていなければ `lark-cli update` を案内。

### 0-3. 認証（デバイスフローに固定・必要権限を初回1回で一括取得）

このスキルは Minutes・VC・ドキュメント・ドライブを使う。**認証はデバイスフロー（QR）に固定**し、
**必要ドメインを最初に全部まとめて要求して、デバイスフローを1回で完結**させる（scope の都度追加でQRを何度も出さない）。

まず認証済みか軽く確認：
```bash
lark-cli minutes +search --owner-ids me --page-size 1 --format json --as user
```
- 成功（このスキルの全機能に必要な権限が既にある）→ 認証OK。0-4 へ。
- 失敗（未認証 / `me` 解決不可 / 権限不足）→ **1回のデバイスフローで一括認証**する：

  1. 必要ドメインをまとめて要求（**これが本スキルの固定の認証コマンド**。`--no-wait --json` がデバイスフロー。エージェント環境ではブロック型は使わない）：
     ```bash
     lark-cli auth login --domain minutes,vc,docs,drive --no-wait --json
     ```
     - `minutes`＝妙記の検索/読取・逐字稿、`vc`＝会議ノート読取、`docs`＝議事録ドキュメント作成/編集、`drive`＝保存先フォルダ作成。**この4ドメインで本スキルの権限は揃う**（scope の個別指定・推測は不要）。
  2. 返却 `verification_url` を **QR画像**にしてユーザーへ提示（「スマホの Lark アプリで読み取ってください」）。`device_code` を控える：
     ```bash
     lark-cli auth qrcode "<verification_url>" --output "lark-login-qr.png"
     ```
  3. ユーザーが「OK／完了」と言ったら、**あなたが**確定：
     ```bash
     lark-cli auth login --device-code "<device_code>"
     ```
  → これで Minutes・VC・docs・drive をまとめて承認するので、**QR読み取りは原則1回**で済む。

  - 例外：万一それでも個別 scope 不足エラーが出た場合のみ、CLI が返す `permission_violations` の不足分を `--scope` で追加（同じくデバイスフロー `--no-wait/--device-code`）。これは通常発生しない想定。
- セキュリティ：トークン/鍵を画面に出さない。`lark-login-qr.png` は完了後に削除してよい。

### 0-4. 保存先フォルダの用意（自動作成がデフォルト）

lark-cli は認証済みなので、**保存先フォルダは Lark Drive に自動作成**してよい。
plan モード的に一言伝えてから実行（書き込み操作のため確認を取る）：
> 「Lark Drive に議事録の保存先フォルダ『議事録』を自動作成して保存先にします。よろしいですか？
> （既存のフォルダを使いたい場合は、そのフォルダの URL を貼ってください）」

- **自動作成（デフォルト）**：
  ```bash
  lark-cli drive +create-folder --name "議事録" --format json --as user
  ```
  返却 JSON の **`folder_token`** を保存先として使う（`--folder-token` を省略するとルート直下に作成される）。
- **既存フォルダを使う場合**：ユーザーが貼った URL から token を抽出（`/folder/` の後ろ、または最後のパスセグメント、`?` 以降は除去）。
- 重複防止：このウィザードは設定ファイルが無いときだけ走るため、通常は一度しか作成しない。再設定時は既存フォルダの再利用も可。

### 0-5. タグの設定（任意）

> 「ファイル名に付ける目印タグはありますか？（例：(議事録)）。なければ『なし』でOK。」

「なし」なら空文字。

### 0-6. 設定の保存

`~/.claude/gijiroku-config.json` に `folder_token` / `tag` / `diagram_ref_doc`（空でよい）を書き込み、
「設定を保存しました。次回からこの手順は不要です」と伝えて Step 1 へ。

---

## Step 1〜: 議事録の作成（毎回）

> 以降、`<FOLDER_TOKEN>` と `<TAG>` は設定ファイルの値を使う。全操作 `--as user`。

### Step 1: 最新 Minutes を取得（自分が所有）

```bash
lark-cli minutes +search --owner-ids me --format json --page-size 10 --as user
```
- `items` を `start_time`（無ければ `create_time`）の**降順**で並べ、最新1件の `token` を取得 → `minute_token`。
- 候補が紛らわしい場合は、上位数件をユーザーに見せて選んでもらってよい。

### Step 2: 逐字稿・要約・タスク・章節を取得

```bash
lark-cli vc +notes --minute-tokens <minute_token> --format json --as user
```
- `transcript`（逐字稿全文）/ `summary` / `todos` / `chapters` を取得（存在するもののみ）。
- 逐字稿はローカルファイル（`artifacts.transcript_file`）に出力されることがある。その場合は Read して全文を取得。

メタ情報：
```bash
lark-cli minutes minutes get --params "{\"minute_token\": \"<minute_token>\"}" --format json --as user
```
→ `title`（会議タイトル）、`url`（録画リンク）、`duration`、`create_time` を記録。

### Step 3: 図解スタイルの決定

- 設定の `diagram_ref_doc` が空でなければ：
  ```bash
  lark-cli docs +fetch --api-version v2 --doc <diagram_ref_doc> --doc-format markdown --as user
  ```
  を読み、最適な図解（最大4種）を選ぶ。
- 空なら、**本スキル末尾の「図解カタログ」**から内容に最適な図解（最大4種）を選ぶ。

### Step 4: 調査タスクの処理（条件付き）

逐字稿・todos に「〜を調べる」「〜について調査する」「〜を確認する」「〜をリサーチする」等が含まれる場合のみ：
1. WebSearch で 1〜3 件調べ、要点を箇条書きにまとめる（Step 6 の「調査結果」セクションで使用）。
該当しなければスキップ。

### Step 5: 今日の日付を取得

```bash
date +%Y-%m%d
```

### Step 6: ドキュメントタイトルを決定

```
フォーマット: YYYY-MMDD<TAG>〈内容要旨〉：〈会議タイトル〉
例: 2026-0627(議事録)新機能方針決定：週次レビュー

ルール:
- YYYY-MMDD は今日の日付
- <TAG> は設定の tag（空なら省略）
- 〈内容要旨〉は10文字以内で内容を一言要約
- ：（全角コロン）の後に会議の元タイトルをそのまま
```

### Step 7: 議事録ドキュメントを作成

`lark-doc-xml.md` と `lark-doc-create-workflow.md` に従い、内容に応じた構成で XML を組み立てる（固定テンプレに無理に合わせない）。基本要素：

```xml
<title>YYYY-MMDD<TAG>〈要旨〉：〈タイトル〉</title>

<h1>📋 議事録</h1>
<p>日時：〈開催日時〉　/　参加者：〈参加者〉　/　<a href="〈録画URL〉">録画リンク</a></p>

<h2>概要</h2>
<p>〈会議の目的・背景・結論を2〜3文で〉</p>

<h2>✅ 決定事項</h2>
<ul><li>〈決定事項〉</li></ul>

<h2>📌 タスク・アクション</h2>
<ul><li><checkbox done="false">〈タスク〉 — 担当: 〈名前〉 / 期限: 〈日付〉</checkbox></li></ul>

<!-- 調査タスクがある場合のみ -->
<h2>🔍 調査結果</h2>
<callout emoji="🔍" background-color="light-blue" border-color="blue">
  <p><b>〈テーマ〉</b></p>
  <ul><li>〈要点〉</li></ul>
</callout>

<h2>📊 図解</h2>
<!-- 内容に応じた図解を最大4点。Mermaid は主 Agent が直接挿入可： -->
<!-- <whiteboard type="mermaid">flowchart TD ...</whiteboard> -->
<!-- 簡単な図は <whiteboard type="svg">完全自包含SVG</whiteboard> -->

<h2>📄 逐字稿（原本）</h2>
<callout emoji="📄" background-color="light-gray" border-color="gray">
  <p><b>以下は自動文字起こし原文です（誤変換を含みます）。</b></p>
</callout>
```

**作成コマンド（長文対策）：**
```bash
# 骨格＋本文を相対パスのファイルに書き、@file で渡す（cwd 配下の相対パスのみ可）
lark-cli docs +create --api-version v2 \
  --parent-token <FOLDER_TOKEN> \
  --content @.gijiroku_body.xml \
  --as user
```
- 逐字稿が長い（5000文字超）場合：本体作成では「📄 逐字稿（原本）」見出し＋注記まで作り、逐字稿本文は別途 append：
  ```bash
  # 逐字稿を XML エスケープ（& < >）して <pre lang="text"><code>…</code></pre> にラップしたファイルを用意し
  lark-cli docs +update --api-version v2 --doc <document_id> --command append --content @.gijiroku_transcript.xml --as user
  ```
- 一時ファイル（`.gijiroku_body.xml` 等）は完了後に削除する。

### Step 8: 完了報告

作成したドキュメントの URL をユーザーに伝える。`lark-cli` の更新通知（`_notice.update`）が出ていれば、作業後に `lark-cli update` を提案する。

---

## 図解カタログ（`diagram_ref_doc` 未設定時に使用）

内容に合うものを最大4種選ぶ。無理に4点用意しない。

| 用途 | 種類 / 記法 |
|------|------------|
| 全体像・構造整理・論点整理 | マインドマップ `<whiteboard type="mermaid">mindmap …</whiteboard>` |
| 手順・意思決定・業務/承認フロー | フローチャート `<whiteboard type="mermaid">flowchart TD …</whiteboard>` |
| 人物/システム間の時系列のやり取り | シーケンス図 `<whiteboard type="mermaid">sequenceDiagram …</whiteboard>` |
| スケジュール・工程 | ガントチャート `<whiteboard type="mermaid">gantt …</whiteboard>` |
| 出来事の時系列 | タイムライン `<whiteboard type="mermaid">timeline …</whiteboard>` |
| 状態の遷移 | 状態遷移図 `<whiteboard type="mermaid">stateDiagram-v2 …</whiteboard>` |
| データ構造・項目の関係 | ER図 `<whiteboard type="mermaid">erDiagram …</whiteboard>` |
| 構成比 | 円グラフ `<whiteboard type="mermaid">pie …</whiteboard>` |
| 優先度・分類（2軸） | 4象限 `<whiteboard type="mermaid">quadrantChart …</whiteboard>` |
| 比較・一覧 | 表 `<table>…</table>`（ネイティブXML） |
| 役割・階層・組織 | 組織図 `<whiteboard type="plantuml">…</whiteboard>` または表 |
| 強調・補足 | カラーパネル `<callout emoji="💡" …>…</callout>` |

> Mermaid のラベルに `<` を使わない（XML と衝突）。`-->` は可。日本語ラベルは `["…"]` で囲むと安全。

## 注意事項
- 全操作は `--as user`。
- ドキュメント作成前に必ず `lark-doc-xml.md` を読む。
- 長文は一度に `--content` に詰め込まず、骨格 → append の2段階で。
- トークン・鍵を画面・ログに出さない。
