---
title: AIコードレビュー結果を「やりっぱなし」にしないために、Cursor スキル + gh で運用フローを作った話
tags:
  - pullrequest
  - コードレビュー
  - cursor
  - スキル
  - ハーネスエンジニアリング
private: false
updated_at: '2026-05-17T13:59:10+09:00'
id: ce77e0d09c425dc2ba7c
organization_url_name: dialog-inc
slide: false
ignorePublish: false
---

## TL;DR

- AI レビューの出力を **GitHub の PR 本文 / 行コメントに残す** までを 1 つのワークフローにまとめたい
- そこで Cursor の「スキル」として `ai-pr-review-tooling` を作成
- ポイントは 4 つ
  - **「投稿しますか？」→ Yes で `gh` を自動実行** する運用ポリシー
  - `gh api` で **PR の行コメントを投稿するシェルスクリプト** を同梱（`headRefOid` 自動取得 / `jq --rawfile` で長文対応）
  - **投稿元フッタ** を単一ソースで管理し、AI 由来のコメントを後から識別可能にする
  - 21 ファイル以上の大型 PR では**段階レビュー**を強制し、「投稿しますか？」も全段揃ったあとに 1 回だけ

---

## 背景：AI レビュー、出して終わりになっていないか

最近は Cursor / Claude Code / CodeRabbit 等で **PR を AI にレビューさせる** ことが当たり前になってきました。一方で実運用では、

- AI レビューの結果を **PR にちゃんと残せていない**（チャットに出して終わり）
- 行コメントとして残すには `gh api` の組み立てが面倒で、毎回手で書いている
- AI 由来のコメントなのか人間の指摘なのか、**後から見て区別がつかない**
- 大きな PR を 1 発で AI に投げると、**コンテキスト溢れて品質が落ちる**

…という地味な摩擦が積み重なります。

弊チームでは Cursor のスキル（`/.cursor/skills/`）で**フロントエンドのセルフレビュー**を回すフローを既に作っていました。問題は「**レビュー結果を GitHub にどう載せるか**」の標準化です。

そこで「**AI レビューの実行は別スキルに任せ、その後段で `gh` を叩いて PR に残すまでを面倒見るスキル**」として `ai-pr-review-tooling` を作りました。

**GitHub Copilot** や **Claude Code Review** とは制約／コスト／カスタマイズ性／自社規約強制力でこのスキルに優位性があると考えています。

---

## 何を作ったか

ディレクトリ構成はシンプルです。

```
.cursor/skills/ai-pr-review-tooling/
├── SKILL.md
└── scripts/
    ├── append-self-review-footer.sh   # 本文末尾にフッタを追記
    ├── post-pr-line-comment.sh        # PR の特定行にレビューコメントを投稿
    └── self-review-footer.md          # 投稿元フッタの単一ソース
```

`SKILL.md` は Cursor のスキルとして読み込まれ、エージェントの行動指針として機能します。役割は次の 3 つだけ。

1. **AI レビュー結果が出たあと**、「GitHub に投稿しますか？」を 1 回だけ聞く
2. **Yes なら `gh` / 同梱スクリプトを自動実行** する（コマンドごとの追加確認はしない）
3. **`docs/workflows/ai-pr-review-tooling.md`**（運用ガイドの単一ソース）のメンテを補助する

具体的なレビュー観点は別スキル（`frontend-review`）に切り出してあり、このスキル自体は **「載せ方」だけ** に責務を絞っています。

---

## 設計のポイント

### 1. 「投稿しますか？」→ Yes で自動実行する

AI に毎回 `gh` のコマンドを出力させ、人間がコピペ実行する運用は、結局**ほぼ全員やらなくなります**。
かといってレビュー結果が出た瞬間に勝手に投稿させると誤投稿が怖い。

そこで `SKILL.md` に次のポリシーを明記しました。

> 1. レビュー結果を出したあと、**「この内容を GitHub に投稿しますか？」** と聞く
> 2. ユーザーが **Yes** と答えたら、**その場で `gh` / スクリプトを実行してよい**。コマンドごとの追加確認は不要
> 3. 実行直前に、**対象 PR・投稿の種類（本文 / 行コメント）・主要な path:line を一行で示してから**走らせる（誤投稿防止）
> 4. **No** と言ったら絶対に `gh` を実行しない

これで、

- ユーザー: Yes / No の判断は **1 回だけ**
- エージェント: 内容のサマリーを 1 行示してから自動実行
- 監査性: コマンドは履歴に残るし、フッタで AI 由来とわかる

…と、安全側に倒しつつ実行コストを最小化できます。

### 2. 行コメント投稿スクリプトで `gh api` の地雷をまとめて踏み抜く

`gh pr review --comment` だと **PR 全体へのコメント**になるので、行単位で残せません。
**特定の行**に紐づくレビューコメントを投稿するには `POST /repos/{owner}/{repo}/pulls/{pull_number}/comments` を叩く必要があります。

これを毎回エージェントに組み立てさせると、

- `commit_id` に何を入れる？ → **`headRefOid` を毎回 `gh pr view --json` で取得**する必要がある
- `body` に複数行 / バッククォート / 引用符が混ざるとシェルエスケープで死ぬ
- `jq --arg` だと長文で **ARG_MAX に当たって**失敗するケースがある

…と地雷だらけ。なので `post-pr-line-comment.sh` を同梱してまとめて吸収しています。

```bash
#!/usr/bin/env bash
set -euo pipefail

PR_NUMBER="${1:?PR_NUMBER required}"
PATH_REL="${2:?path required}"
LINE="${3:?line required}"
BODY_FILE="${4:-/dev/stdin}"

SCRIPT_DIR=$(cd "$(dirname "$0")" && pwd)
FOOTER="${SCRIPT_DIR}/self-review-footer.md"

tmp_body=$(mktemp)
tmp_json=$(mktemp)
trap 'rm -f "$tmp_body" "$tmp_json"' EXIT

cat "$BODY_FILE" >"$tmp_body"
# 末尾改行なしだとフッタが直前行に続いて Markdown が崩れるので 1 行挟む
printf '\n' >>"$tmp_body"
cat "$FOOTER" >>"$tmp_body"

REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
OWNER="${REPO%%/*}"
NAME="${REPO##*/}"
SHA=$(gh pr view "${PR_NUMBER}" --json headRefOid -q .headRefOid)

# 長文本文は --arg ではなく --rawfile で渡し ARG_MAX を避ける
jq -n \
  --rawfile body "$tmp_body" \
  --arg commit "${SHA}" \
  --arg path "${PATH_REL}" \
  --argjson line "${LINE}" \
  '{body: $body, commit_id: $commit, path: $path, line: $line}' \
  >"$tmp_json"

gh api "repos/${OWNER}/${NAME}/pulls/${PR_NUMBER}/comments" --input "$tmp_json"
```

使い方は次のとおりで、エージェントもユーザーもこれだけ覚えれば OK。

```bash
# stdin で渡す（短い指摘向け）
echo $'**critical** apiFetch を使うこと\n\n（理由: …）' | \
  .cursor/skills/ai-pr-review-tooling/scripts/post-pr-line-comment.sh \
    42 frontend/apps/web/src/foo/Foo.tsx 12

# ファイルで渡す（複数行 / 引用符あり）
.cursor/skills/ai-pr-review-tooling/scripts/post-pr-line-comment.sh \
  42 frontend/apps/web/src/foo/Foo.tsx 12 comment.md
```

`SKILL.md` 側にも「**行コメントが必要で Yes が得られたら、このスクリプトを実行（手で `gh api` を組み立てない。エスケープ漏れ防止）**」と書いてあるので、エージェントは素直にスクリプト経由で投稿します。

### 3. 投稿元フッタを単一ソースで管理

AI 由来のコメントは、後から「これって人間が書いたんだっけ？」と判別不能になりがちです。
そこで本文末尾に**目立たない識別子**を必ず付けるルールにしました。

`scripts/self-review-footer.md`:

```markdown

---
<sub>Cursor self-review · frontend-review · ai-pr-review-tooling</sub>
```

`append-self-review-footer.sh` は `cat` するだけのほぼ 1 行スクリプトですが、重要なのは**手で書かせないこと**。
スクリプト経由でしか付与されないようにすることで、

- 文言を変えたいときは `self-review-footer.md` を 1 箇所直すだけ
- 「フッタを忘れた / 微妙に違う文言で書いた」事故を防げる
- GitHub の検索で `Cursor self-review` を検索すれば AI 由来のコメントだけが拾える

…という運用上のメリットが出ます。

PR 本文に貼るときも、

```bash
.cursor/skills/ai-pr-review-tooling/scripts/append-self-review-footer.sh \
  < body.md > body-with-footer.md
gh pr create --title "feat: ..." --body-file body-with-footer.md
# 既存 PR を上書きするとき
gh pr edit <PR_NUMBER> --body-file body-with-footer.md
```

このように、**本文→フッタ追記→`gh`** の 3 ステップを必ず通すようにしてあります。

---

## 大きな PR は段階レビューを強制する

LLM のコンテキストは有限なので、変更ファイルが多い PR を 1 回で投げるとレビュー品質が露骨に落ちます。
そこで `SKILL.md` には次のルールも入れてあります。
(もちろん、なるべく細かい間隔でRVを行うことをおすすめします。)

> 対象が **21 ファイル以上** のときは、
> `frontend-review` の**段階レビュー**に従う。
> `gh` の「投稿しますか？」は **全段のレビュー出力が揃ったあとに 1 回**だけ。

「21」という閾値は、過去の PR でレビュー品質が落ち始めた肌感に合わせた数字です。
重要なのは「**段階ごとに投稿確認を挟まない**」というルールで、

- 段ごとに「投稿しますか？」を聞くと UX が地獄になる
- 段の途中で投稿してしまうと、後から差分を全体最適できない

ので、**最後に 1 回だけまとめて投稿**する形に統一しています。

---

## SKILL.md と運用ドキュメントの責務分担

`SKILL.md` は AI エージェントに毎回読み込まれるので、長文化するとプロンプトコストが膨らみます。
そこで次の責務分担にしました。

| 役割 | 配置 | 理由 |
|------|------|------|
| エージェント行動指針（短く・固定） | `.cursor/skills/ai-pr-review-tooling/SKILL.md` | 毎回読み込まれる前提で簡潔に保つ |
| 比較表 / 公式リンク / メンテ手順 | `docs/workflows/ai-pr-review-tooling.md` | 時点情報が混ざるので人間が四半期で見直す |
| レビュー観点（フロント） | `.cursor/skills/frontend-review/SKILL.md` | 「載せ方」と「観点」は別物として切り離す |
| 機械向け規約 | `.cursor/rules/*.mdc` | エディタが自動適用するルール |
| 規約本文（人間向け） | `docs/code-rule/` | レビューで参照する一次資料 |

`SKILL.md` には **「重複禁止」** という項目を入れて、CodeRabbit の料金やセキュリティ細目など**時点で古くなる情報**は、エージェントの回答に書かせず**ドキュメントへ誘導**させるようにしています。
これだけでチャット内に古い情報が拡散するのをかなり防げます。

---

## CodeRabbit との役割分担

「それ CodeRabbit でよくない？」という疑問は当然あるので、ポジションを整理しておきます。

それはそうです。。。w
すでに社内で使えるようになっているCursorやClaudeでどこまでできるかを確認したかったからです。

| ステップ | 担当 |
|----------|------|
| 1. セルフレビュー（TL;DR / `path:line` / critical / warning / nit を出す） | `frontend-review` スキル |
| 2. GitHub に載せる（PR 本文 + 行コメント） | `ai-pr-review-tooling` スキル + `gh` |
| 3. push のたびに完全自動で再実行 | CI / 外部 SaaS（CodeRabbit 等）の領域 |

つまり今回作ったスキルは「**マージ前の手動セルフ RV + 1 回 `gh`**」までを標準化するもの。
CodeRabbit のような完全自動 bot は別軸として、自チームのリポジトリ規約（命名、レイヤ依存、PSR-12 + 2 スペース等）を**自前ルールで強制したい**領域で機能します。

---

## 実際の使い心地

導入後の体感としては、

- Cursor チャットで「レビューして」→ レビュー出る → 「GitHub に投稿しますか？」→ **Yes** → 行コメントが PR に並ぶ、までが 30 秒
- 後から PR を見返したとき、`Cursor self-review` フッタで **AI 由来コメントだけを目で追える**

逆に、**毎 push の自動レビュー**は仕組み上できないので、そこが欲しい場合は CodeRabbit 等の併用が必要、という割り切りです。

---

## PR を出す前のチェックリスト

`SKILL.md` の末尾にはチェックリストも書いてあります。これがエージェントの最終確認用です。

- [ ] フロント変更なら `frontend-review` で **TL;DR + path:line + critical/warning/nit** まで出したか
- [ ] 対象が **21 ファイル以上**なら **段階レビュー（原則 2 段）**を踏んだか
- [ ] GitHub 投稿する場合は **「投稿しますか？」→ Yes** を得たか
- [ ] 行単位で残す指摘は `path` / `line` / `commit_id`（スクリプトが自動取得）を用意したか

---

## まとめ

- AI レビューは「**実行**」より「**結果をどう残すか**」のほうが運用上の摩擦が大きい
- Cursor スキル + `gh` + 同梱シェルスクリプトという地味な組み合わせだけでも、**「やりっぱなし」を防ぐ標準フロー**は作れる
- ポイントは
  - **Yes 1 回で自動実行する** 行動ポリシーをスキルに書く
  - **`gh api` の地雷はスクリプトに閉じ込める**（`headRefOid` / `jq --rawfile` / フッタ追記）
  - **AI 由来かどうかをフッタで識別可能にする**
  - **「載せ方」と「観点」をスキルとして分割**し、それぞれを短く保つ
- 大型 PR は **21 ファイル閾値で段階レビュー**を強制し、投稿は最後にまとめて 1 回

「AI レビューを導入したけど PR にちゃんと残せていない」というチームには、Cursor スキルに限らず、**Claude Code / Gemini  等でも同じ構造**を真似できると思います。`gh` を叩く部分はどのエージェントから呼んでも同じなので、**スクリプトとフッタの単一ソース化だけ先に整える**のがおすすめです。

## 最後に
本番用途では追加で下記のケースの検証が必要です。
post-pr-line-comment.sh のエラー処理が甘いのでgh 認証エラー、PR が closed、対象 path がdiff外、などのケースで失敗します。
1年後には自信を持ってapmとかで配布できるようになりたいなーと漠然と考えてはいます。
