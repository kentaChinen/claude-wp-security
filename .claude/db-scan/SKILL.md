---
name: db-scan
description: WordPress の mysqldump ファイル（.sql）をスキャンして、マルウェア・不審な管理者アカウント・スクリプト注入・設定改ざんを検出する。サーバー移行前のDB検査として使用する。
argument-hint: [.sqlファイルのパス（例: ~/db-backup.sql）]
---

あなたは WordPress データベースセキュリティの専門家です。
指定された SQL ダンプファイルを検査し、マルウェア・改ざん・不審なデータを日本語でレポートしてください。

**対象ファイル：** $ARGUMENTS

> 注意: このスキルは SQL ダンプファイル（.sql）を対象とします。ライブ DB への直接接続は行いません。

---

## 事前確認

```bash
# ファイルの存在・サイズ確認
ls -lh "$ARGUMENTS"
wc -l "$ARGUMENTS"

# WordPress テーブルが含まれているか確認
grep -c "CREATE TABLE" "$ARGUMENTS" 2>/dev/null
grep "CREATE TABLE" "$ARGUMENTS" | head -20
```

テーブルプレフィックスを確認する（`wp_` 以外の場合は以降のコマンドを調整）：
```bash
grep "CREATE TABLE" "$ARGUMENTS" | head -5
```

---

## Step 1: wp_options の検査（最重要）

`wp_options` にはサイトURL・アクティブプラグイン・ウィジェット設定が含まれる。
マルウェアが最初に改ざんするテーブル。

### 1-1: サイト URL の確認

```bash
SQL="$ARGUMENTS"
grep -A2 "'siteurl'" "$SQL" | head -10
grep -A2 "'home'" "$SQL" | head -10
```

**正常:** 自分のドメインが設定されている
**異常:** 見知らぬドメイン・IP アドレスが設定されている

### 1-2: アクティブプラグインの確認

```bash
SQL="$ARGUMENTS"
grep -A5 "'active_plugins'" "$SQL" | head -30
```

インストールした覚えのないプラグインが含まれていないか確認する。

### 1-3: ウィジェット・テーマ設定へのスクリプト注入

```bash
SQL="$ARGUMENTS"

echo "=== widget_text へのスクリプト注入 ==="
grep -i "widget_text\|widget_block\|widget_custom_html" "$SQL" | \
  grep -i "<script\|javascript:\|eval(\|base64_decode\|document\.write\|atob(" | head -20

echo "=== theme_mods へのスクリプト注入 ==="
grep -i "theme_mods\|_transient\|_theme_" "$SQL" | \
  grep -i "<script\|javascript:\|eval(\|base64_decode" | head -20

echo "=== wp_options 全体への危険パターン注入 ==="
grep -i "<script[^>]*src\|javascript:\|eval(\|base64_decode\|document\.write\|String\.fromCharCode\|atob(\|unescape(" "$SQL" | \
  grep -v "^--" | head -30
```

### 1-4: 不審な自動ロードオプション

```bash
SQL="$ARGUMENTS"

echo "=== 不審な autoload オプション ==="
# URLを含む autoload オプションで、wp 標準以外のもの
grep "'yes'" "$SQL" | grep -i "http\|script\|eval\|base64" | head -20

echo "=== cron ジョブの確認（不審なタスクが登録されていないか） ==="
grep -A10 "'cron'" "$SQL" | head -40
```

---

## Step 2: wp_users の検査

攻撃者が管理者アカウントを追加するケースが多い。

```bash
SQL="$ARGUMENTS"

echo "=== 全ユーザーの一覧 ==="
grep "^INSERT INTO.*wp_users" "$SQL" | \
  sed "s/),(/\n/g" | \
  grep -oE "\([0-9]+,'[^']+','[^']+','[^']+','[^']+'" | \
  awk -F"'" '{print "ID:"$1" | user_login:"$2" | user_email:"$4}' | head -30
```

確認ポイント：
- 知らないユーザーが存在しないか
- 登録日時（`user_registered`）が感染疑い時期と一致するアカウントがないか
- メールアドレスが見知らぬドメインのアカウントがないか

---

## Step 3: wp_usermeta の検査（管理者権限の確認）

```bash
SQL="$ARGUMENTS"

echo "=== administrator 権限を持つユーザーメタ ==="
grep "s:13:\"administrator\";b:1\|s:13:\"administrator\";s:1:\"1\"\|\"administrator\";b:1" "$SQL" | head -20

echo "=== 全ユーザーの capabilities 確認 ==="
grep "wp_capabilities\|_capabilities" "$SQL" | head -30
```

---

## Step 4: wp_posts の検査

投稿・ページの本文にスクリプトが注入されているケース。

```bash
SQL="$ARGUMENTS"

echo "=== 投稿本文への eval / base64 注入 ==="
grep -i "eval(\|base64_decode(\|base64_encode(" "$SQL" | \
  grep -v "^--\|CREATE\|INSERT INTO.*wp_users\|INSERT INTO.*wp_options" | head -20

echo "=== 投稿本文への <script> 注入 ==="
grep -i "<script" "$SQL" | \
  grep -v "^--\|CREATE\|INSERT INTO.*wp_users" | head -20

echo "=== 外部ドメインへの不審な iframe ==="
grep -i "<iframe" "$SQL" | \
  grep -v "^--\|youtube\|vimeo\|maps\.google" | head -20

echo "=== document.write / String.fromCharCode による難読化 ==="
grep -i "document\.write\|String\.fromCharCode\|atob(\|unescape(" "$SQL" | \
  grep -v "^--\|CREATE" | head -20

echo "=== href / src に外部スクリプトURL ==="
grep -iE "src=['\"]https?://[^'\"]*\.js['\"]" "$SQL" | \
  grep -v "^--\|CREATE" | head -20
```

---

## Step 5: wp_postmeta の検査

```bash
SQL="$ARGUMENTS"

echo "=== postmeta への危険パターン注入 ==="
grep -i "eval(\|base64_decode\|<script\|javascript:" "$SQL" | \
  grep "wp_postmeta" | head -20
```

---

## Step 6: wp_comments の検査（スパム・注入）

```bash
SQL="$ARGUMENTS"

echo "=== コメントへのスクリプト注入 ==="
grep -i "<script\|javascript:\|eval(\|base64_decode" "$SQL" | \
  grep "wp_comments" | head -20

echo "=== 承認済みコメントへの外部リンク大量注入 ==="
grep "wp_comments.*approved" "$SQL" | grep -c "http" 2>/dev/null || true
```

---

## Step 7: 隠れたバックドアユーザーの確認

```bash
SQL="$ARGUMENTS"

echo "=== 最近登録されたユーザー（登録日でソート） ==="
grep "^INSERT INTO.*wp_users" "$SQL" | \
  grep -oE "'[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9:]+'" | sort -r | head -10

echo "=== メールアドレスのドメイン一覧 ==="
grep "^INSERT INTO.*wp_users" "$SQL" | \
  grep -oE "'[^']+@[^']+'" | sort -u | head -20
```

---

## Step 8: DB 全体への包括的パターンスキャン

```bash
SQL="$ARGUMENTS"

echo "=== str_rot13 / hex decode パターン ==="
grep -i "str_rot13\|hex2bin\|chr(.*ord\|pack.*H\*" "$SQL" | \
  grep -v "^--\|CREATE" | head -20

echo "=== 外部 URL（http/https）を含む不審なオプション ==="
grep -iE "https?://[a-z0-9-]+\.[a-z]{2,}/[a-z0-9_/-]+\.(php|exe|sh|py)" "$SQL" | \
  grep -v "^--\|CREATE\|readme\|license\|changelog\|github\.com\|wordpress\.org\|gravatar\.com\|w\.org" | head -20

echo "=== SQL インジェクション痕跡 ==="
grep -i "UNION.*SELECT\|DROP TABLE\|TRUNCATE TABLE\|DELETE FROM\|UPDATE.*SET.*WHERE 1=1" "$SQL" | \
  grep -v "^--\|^/\*" | head -10
```

---

## レポート形式

```markdown
## DB スキャンレポート

**対象ファイル:** [パス]
**スキャン日時:** [日時]
**テーブルプレフィックス:** wp_

### 発見された問題

#### 🔴 即座に対応が必要
| テーブル | 内容 | 対応 |
|---------|------|------|
| wp_users | 不審な管理者アカウント [username] | アカウント削除 |
| wp_options | siteurl が外部ドメインに書き換え | 正しいURLに修正 |

#### 🟡 要確認
| テーブル | 内容 |
|---------|------|
| wp_posts | ID:XX の本文に <script> タグ |

#### ✅ 問題なし
（問題のなかった項目）

### ユーザーアカウント一覧

| ID | ユーザー名 | メール | 登録日 | 評価 |
|----|----------|--------|--------|------|
| 1  | admin    | ... | 2023-xx-xx | ✅ 正常 |

### wp_options 重要設定

| オプション | 値 | 評価 |
|-----------|-----|------|
| siteurl | https://... | ✅ 正常 |
| home | https://... | ✅ 正常 |
| active_plugins | ... | ✅/⚠️ |

### 推奨対応

1. （問題があれば対応手順を記載）
2. 不審ユーザーは C サーバーインポート後に WordPress 管理画面から削除
3. スクリプト注入が見つかった投稿は内容を確認・修正してからインポート
```

---

## 問題が見つかった場合の対処

### 不審ユーザーの削除（SQL で直接修正）

```sql
-- スキャン結果で特定した不審ユーザーを削除
-- （インポート前にSQLファイルを直接編集する）
DELETE FROM wp_users WHERE user_login = '不審なユーザー名';
DELETE FROM wp_usermeta WHERE user_id = [不審なユーザーのID];
```

### wp_options の URL 修正（SQL で直接修正）

```sql
-- サイトURLが改ざんされていた場合
UPDATE wp_options SET option_value = 'https://正しいドメイン.com' WHERE option_name = 'siteurl';
UPDATE wp_options SET option_value = 'https://正しいドメイン.com' WHERE option_name = 'home';
```

### 投稿への注入コード削除

注入が見つかった場合は、C サーバーインポート後に WordPress 管理画面の投稿編集で該当コードを手動削除するか、SQL ファイルをテキストエディタで修正してからインポートする。
