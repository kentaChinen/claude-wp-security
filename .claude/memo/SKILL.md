---
name: memo
description: 検査中に発見したコードパターン・IOC・攻撃手法・メモをナレッジベース（.claude/threat-intel/）に記録・検索・管理する。add/show/search/promote の4操作に対応。
argument-hint: [add <type> <内容> | show [type] | search <キーワード> | promote]
---

検査中に発見した情報をナレッジベース（`.claude/threat-intel/`）に蓄積・管理します。
蓄積した知識は次回以降の検査で自動参照され、見落としを防ぎます。

## ナレッジベースの構成

| ファイル | 種別 | 用途 |
|---------|------|------|
| `.claude/threat-intel/patterns.md` | `pattern` | 悪意あるコードパターン（eval・base64等） |
| `.claude/threat-intel/ioc.md` | `ioc` | IOC（URL・IP・ドメイン・ファイルハッシュ） |
| `.claude/threat-intel/techniques.md` | `technique` | 攻撃手法・感染経路 |
| `.claude/threat-intel/notes.md` | `note` | その他メモ・気になる事項 |

---

## コマンド

引数 `$ARGUMENTS` の先頭の単語でコマンドを判定する。
コマンドが省略された場合や不明な場合は `show` として扱う。

---

### `add` — メモを追記する

**構文:** `/memo add <type> <内容>`

**手順:**

1. `$ARGUMENTS` を解析して type（pattern/ioc/technique/note）と内容を取得する
2. type に対応するファイルを確認する:
   - ファイルが存在しない → 後述の「初期化テンプレート」で Write ツールを使って作成する
   - ファイルが存在する → Read ツールで読み込む
3. 現在日時（`date +%Y-%m-%d`）を取得する
4. Edit ツールで以下のフォーマットで追記する

**追記フォーマット:**

`patterns.md` の場合（テーブル末尾に1行追加）:
```
| {YYYY-MM-DD} | `{コードパターン}` | {説明} | {発見場所（任意）} |
```

`ioc.md` の場合（テーブル末尾に1行追加）:
```
| {YYYY-MM-DD} | `{IOC値}` | {URL / IP / Domain / Hash / FileName} | {詳細} |
```

`techniques.md` の場合（ファイル末尾にブロック追加）:
```

## {攻撃手法のタイトル}（{YYYY-MM-DD} 記録）

{内容の詳細説明}

- **発見場所:** {パス（任意）}
- **関連パターン:** {関連するパターン（任意）}
```

`notes.md` の場合（ファイル末尾にブロック追加）:
```

## {タイトル}（{YYYY-MM-DD} 記録）

{メモの内容}
```

5. 追記完了後、追記した内容を引用して「✅ 記録しました」と報告する
6. 重要度が高い発見の場合（マルウェア確定・新しい手口等）は「`/memo promote` で各ファイルに昇格できます」と案内する

---

### `show` — メモを一覧表示する

**構文:** `/memo show [type]`

**手順:**

1. type が指定されていない場合: 全4ファイルを Read ツールで読み込んで表示する
2. type が指定されている場合: 対応するファイルのみ Read ツールで読み込んで表示する
3. 各ファイルが存在しない場合は「まだ記録がありません」と表示する
4. 表示後に「`/memo search <キーワード>` で絞り込み検索ができます」と案内する

---

### `search` — キーワード検索

**構文:** `/memo search <キーワード>`

**手順:**

```bash
THREAT_DIR="$(pwd)/.claude/threat-intel"
KEYWORD=$(echo "$ARGUMENTS" | sed 's/^search[[:space:]]*//')

echo "=== ナレッジベース検索: $KEYWORD ==="
grep -rn "$KEYWORD" "$THREAT_DIR" 2>/dev/null || echo "該当するメモが見つかりませんでした"
```

結果をファイル別にグループ化して見やすく整形して表示する。

---

### `promote` — CLAUDE.md・スキルファイルへ昇格

**構文:** `/memo promote`

蓄積したメモの中から重要な発見を、CLAUDE.md や各スキルファイルへ統合する。

**手順:**

1. 全4ファイルを Read ツールで読み込む
2. `CLAUDE.md` を Read ツールで読み込む
3. `malware-scan/SKILL.md` を Read ツールで読み込む
4. 以下の昇格候補を特定してユーザーに提案する:

| 発見の種別 | 昇格先 | 昇格条件 |
|-----------|--------|---------|
| 確認済みマルウェアファイル名 | CLAUDE.md「既知のマルウェア」テーブル | 脅威レベルが「高危険」以上 |
| 新しいコードパターン | malware-scan/SKILL.md の grep パターン | 既存パターンに含まれていない |
| 新しい攻撃手法 | security-audit/SKILL.md の参照情報 | 新手口・希少な手法 |
| IOC（URL/IP/Domain） | CLAUDE.md「既知のマルウェア」テーブルの備考 | 外部C2サーバー等の確認済み IOC |

5. 各昇格候補について「昇格しますか？」と確認を取る
6. 承認された項目のみ Edit ツールで各ファイルに追記する

**CLAUDE.md「既知のマルウェア」テーブルへの追記フォーマット:**
```
| ファイル名またはパターン | 種別（バックドア/Webシェル等） | 脅威レベル |
```

**malware-scan/SKILL.md への追記フォーマット:**
```bash
echo "=== [ナレッジベース由来] {パターン名} ==="
grep -rn "{grepパターン}" "$TARGET_DIR" --include="*.php" 2>/dev/null | grep -v "vendor/"
```

---

## 初期化テンプレート

ファイルが存在しない場合、Write ツールで以下のテンプレートを使って作成すること。

### `patterns.md`
```markdown
# 悪意あるコードパターン

検査中に発見した危険なコードパターン。`/memo add pattern` で追記。
スキャン時はこのパターンを優先的に grep で検索すること。

| 日付 | パターン | 説明 | 発見場所 |
|------|---------|------|---------|
```

### `ioc.md`
```markdown
# Indicators of Compromise (IOC)

発見した悪意あるURL・IP・ドメイン・ファイルハッシュ。`/memo add ioc` で追記。
スキャン時は grep でこれらの値の混入を確認すること。

| 日付 | IOC | 種別 | 詳細 |
|------|-----|------|------|
```

### `techniques.md`
```markdown
# 攻撃手法・感染経路

発見した攻撃手法や感染経路の記録。`/memo add technique` で追記。
```

### `notes.md`
```markdown
# メモ・気になる事項

検査中の気になる点や後で確認すべき事項。`/memo add note` で追記。
```
