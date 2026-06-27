# Gijiroku プラグイン（Lark 議事録 自動生成）

Lark の録画（Minutes / 妙記）から、要約・タスク・図解入りの議事録ドキュメントを自動生成する Claude Code プラグインです。
**テナント固有の値（保存先フォルダ・認証）は初回のチャット上ウィザードで設定**します。プラグイン本体にはトークン等を一切含みません。

> 📖 受け手向けの詳しい使い方・FAQ は **[使い方.md](./使い方.md)** を参照してください。

---

## 受け手の導入（ターミナル不要・2コマンド）

Claude Code の中で次を実行するだけ：

```
/plugin marketplace add momiji-project/gijiroku-marketplace
/plugin install gijiroku@gijiroku-marketplace
```

あとは `/Gijiroku-my` と打つと、**初回だけ**セットアップウィザードが起動します。
画面の質問に答えるだけ（コマンド操作は AI が代行）：

1. （未導入なら）`lark-cli` の導入に同意 ※Node.js/npm が必要
2. **QRコードを1回だけスマホの Lark アプリで読み取る** → 「OK」（必要権限はこの1回でまとめて許可：minutes / vc / docs / drive）
3. 保存先フォルダの **自動作成に同意**（既存フォルダを使う場合のみ URL を貼る）
4. ファイル名タグ（任意）を答える

→ 2回目以降は `/Gijiroku-my` だけで最新の録画から議事録が作られます。

### 前提
- Claude Code（インストール済み）
- `git`（マーケットプレイス取得時に使用。Mac は初回に導入を促されることあり）
- Node.js / npm（`lark-cli` 用。ウィザードが導入を案内）

---

## 配布する人の公開手順（一度きり）

このフォルダ（`gijiroku-marketplace/`）を **公開** GitHub リポジトリに push するだけです。

> ⚠️ **プライバシー**：`git commit` の名前・メールは公開されます。本名・常用メールを出したくない場合は、
> 下記のとおり **リポジトリ専用の匿名 identity** を先に設定してください（GitHub の noreply メール推奨）。
> また `marketplace.json` の `owner.name` を任意のハンドル名に置き換えてください。

```bash
# 1) marketplace.json の owner.name を編集（REPLACE_WITH_YOUR_HANDLE_OR_ORG → 好きなハンドル）

# 2) このフォルダで初期化（cd は環境に合わせて）
cd gijiroku-marketplace
git init

# 3) このリポジトリ専用の匿名 identity（本名・常用メールを出さない）
git config user.name  "your-handle"
git config user.email "your-handle@users.noreply.github.com"

# 4) コミット
git add .
git commit -m "Add gijiroku plugin"

# 5) 公開リポジトリを作成して push
#    gh CLI がある場合（最短）:
gh repo create gijiroku-marketplace --public --source=. --remote=origin --push
#    gh が無い場合: GitHub で空の public リポジトリを作成してから
# git remote add origin https://github.com/<your-account>/gijiroku-marketplace.git
# git branch -M main && git push -u origin main
```

受け手には、上の「受け手の導入」2コマンドの `<github-owner>/<repo>` を埋めて伝えてください。

### 更新の配り方
`plugins/gijiroku/.claude-plugin/plugin.json` の `version` を上げて push すると、受け手に更新が届きます。

---

## 構成

```
gijiroku-marketplace/
├── .claude-plugin/marketplace.json     # マーケットプレイス定義
├── plugins/gijiroku/
│   ├── .claude-plugin/plugin.json      # プラグイン定義（version で更新管理）
│   └── skills/Gijiroku-my/SKILL.md     # 本体（初回ウィザード＋図解カタログ内蔵）
├── .gitignore                          # ローカル設定/一時ファイルを公開しない
└── README.md
```

## 設計メモ（プライバシー）
- テナント固有値（保存フォルダ token・図解参照 doc・ドメイン・氏名・メール 等）は**コードに含めない**。
- 各受け手の値は、その人の `~/.claude/gijiroku-config.json` にローカル保存（リポジトリには入らない）。
- 認証は各受け手が自分の Lark アカウントで実施（`lark-cli auth login`、QR）。
