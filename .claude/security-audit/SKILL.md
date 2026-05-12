---
name: security-audit
description: WordPress サイト全体のセキュリティを網羅的に検査する総合監査スキル。マルウェア・脆弱性・設定不備・パーミッション・.htaccess を全フェーズで漏れなくチェックし、進捗ファイルで未検査項目を追跡する。「検査」「検索」「調査」「調べて」「確認」「チェック」「スキャン」キーワードで自動起動する。
argument-hint: [resume=前回の続きから | full=全フェーズ強制実行（デフォルト: full）]
---

あなたは WordPress セキュリティ監査の専門家です。サイト全体を **漏れなく** 検査し、検査状況を追跡しながら進めてください。

## 監査の進め方

1. まず `.security-audit-progress.md` が存在するか確認する
   - 存在する場合: 未完了項目を確認してユーザーに「前回の続きから再開しますか？」と聞く
   - 存在しない場合: Phase 0 から開始する
2. 各フェーズを順番に実行し、完了したらチェックリストを更新する
3. 全フェーズ完了後に総合レポートを出力する

---

## Phase 0: 監査準備（ファイル棚卸し）

### 0-1: WP ルートの特定

```bash
WP_ROOT="$(realpath "$(pwd)/../../../" 2>/dev/null)"
echo "WP ルート: $WP_ROOT"
ls "$WP_ROOT"
```

### 0-2: WordPress バージョンの確認

```bash
WP_ROOT="$(realpath "$(pwd)/../../../" 2>/dev/null)"
grep "wp_version = " "$WP_ROOT/wp-includes/version.php" 2>/dev/null | head -3
```

### 0-3: PHP ファイルの全棚卸し（ディレクトリ別件数）

```bash
WP_ROOT="$(realpath "$(pwd)/../../../" 2>/dev/null)"

echo "=== ディレクトリ別 PHP ファイル数 ==="
echo "wp-root (直下):       $(find "$WP_ROOT" -maxdepth 1 -name "*.php" | wc -l | tr -d ' ') files"
echo "wp-admin/:            $(find "$WP_ROOT/wp-admin" -name "*.php" | wc -l | tr -d ' ') files"
echo "wp-includes/:         $(find "$WP_ROOT/wp-includes" -name "*.php" | wc -l | tr -d ' ') files"
echo "wp-content/themes/:   $(find "$WP_ROOT/wp-content/themes" -name "*.php" | wc -l | tr -d ' ') files"
echo "wp-content/plugins/:  $(find "$WP_ROOT/wp-content/plugins" -name "*.php" | wc -l | tr -d ' ') files"
echo "wp-content/uploads/:  $(find "$WP_ROOT/wp-content/uploads" -name "*.php" | wc -l | tr -d ' ') files"
echo "wp-content/ (直下):   $(find "$WP_ROOT/wp-content" -maxdepth 1 -name "*.php" | wc -l | tr -d ' ') files"
echo "合計:                 $(find "$WP_ROOT" -name "*.php" | wc -l | tr -d ' ') files"
```

### 0-4: 最近変更されたファイル（30日以内、コア除く）

```bash
WP_ROOT="$(realpath "$(pwd)/../../../" 2>/dev/null)"

echo "=== 最近変更されたファイル（非コア） ==="
find "$WP_ROOT" -mtime -30 -type f \
  \( -name "*.php" -o -name "*.js" -o -name "*.htaccess" -o -name "*.html" \) \
  -not -path "*/wp-includes/*" \
  -not -path "*/wp-admin/*" \
  -ls 2>/dev/null | sort -k8,9
```

### 0-5: 疑わしいファイル名（ランダム文字列 .php）

```bash
WP_ROOT="$(realpath "$(pwd)/../../../" 2>/dev/null)"

echo "=== ランダム名の PHP ファイル（マルウェアの特徴） ==="
find "$WP_ROOT" -name "*.php" \
  -not -path "*/wp-includes/*" \
  -not -path "*/wp-admin/*" \
  -not -path "*/vendor/*" | while read f; do
  basename "$f" | grep -qE '^[a-z0-9]{4,12}\.php$' && \
  basename "$f" | grep -qvE '^(index|header|footer|functions|style|page|single|archive|template|sidebar|search|404|comments|content|loop|entry|post|category|tag|author|date|attachment|image|home|front|blog|shop|cart|checkout|account|widget|plugin|admin|class|core|init|load|ajax|api|theme|config|setup|install|update|upgrade|cron|mail|feed|sitemap|robots|login|register|activate|deactivate|uninstall|upgrade|about|readme|license|sample)\.php$' && \
  echo "$f"
done
```

### 0-6: 進捗ファイルの初期化

以下の内容で `.security-audit-progress.md` を Write ツールで作成する：

```markdown
# セキュリティ監査進捗 — [開始日時]

## ステータス凡例
- ✅ 完了（問題なし）
- ⚠️ 完了（要注意事項あり）
- ❌ 完了（問題検出）
- ⏳ 未実施

## 監査チェックリスト

### Phase 0: 準備・棚卸し
- [ ] 0-1: WP ルート特定
- [ ] 0-2: WordPress バージョン確認
- [ ] 0-3: PHP ファイル棚卸し（件数確認）
- [ ] 0-4: 最近変更されたファイル確認
- [ ] 0-5: 疑わしいファイル名チェック

### Phase A: WordPress ルート直下ファイル検査
- [ ] A-1: ルート直下 PHP ファイルの一覧・内容確認
- [ ] A-2: wp-config.php の安全性確認
- [ ] A-3: wp-blog-header.php の内容確認
- [ ] A-4: wp-login.php の行数・末尾確認
- [ ] A-5: wp-settings.php の末尾確認
- [ ] A-6: index.php の内容確認
- [ ] A-7: wp-cron.php の末尾確認

### Phase B: テーマ検査（全テーマ）
- [ ] B-1: インストール済みテーマ一覧
- [ ] B-2: 各テーマの危険パターンスキャン
- [ ] B-3: 各テーマ内のランダムファイル名チェック
- [ ] B-4: テーマ .htaccess の確認

### Phase C: プラグイン検査（全プラグイン）
- [ ] C-1: インストール済みプラグイン一覧
- [ ] C-2: 各プラグインの危険パターンスキャン
- [ ] C-3: プラグイン内のランダムファイル名チェック

### Phase D: wp-content 直下・uploads 検査
- [ ] D-1: wp-content 直下の不審ファイル確認
- [ ] D-2: `/uploads-scan` で uploads を詳細検査（PHP混入・偽装・SVG・最近変更ファイル・.htaccess）

### Phase E: WordPress コア整合性確認
- [ ] E-1: wp-includes 内の不審パターン検索
- [ ] E-2: wp-admin 内の不審パターン検索
- [ ] E-3: コアファイル改ざんの基本確認

### Phase F: .htaccess 全箇所確認
- [ ] F-1: WP ルート .htaccess
- [ ] F-2: テーマ .htaccess
- [ ] F-3: uploads .htaccess
- [ ] F-4: その他 .htaccess（プラグイン等）

### Phase G: ファイルパーミッション
- [ ] G-1: 他者書き込み可能なファイル・ディレクトリ
- [ ] G-2: PHP ファイルの実行ビット
- [ ] G-3: wp-config.php のパーミッション
- [ ] G-4: .htaccess のパーミッション

### Phase H: 設定・認証情報確認
- [ ] H-1: wp-config.php シークレットキーの強度
- [ ] H-2: wp-config.php デバッグモード設定
- [ ] H-3: wp-config.php テーブルプレフィックス
- [ ] H-4: xmlrpc.php のアクセス可否
- [ ] H-5: /wp-json/wp/v2/users の無効化・著者アーカイブ無効化
- [ ] H-6: WAF の導入状況

### Phase I: データベース検査（移行前に .sql ファイルがある場合）
- [ ] I-1: wp_options の siteurl / home 確認
- [ ] I-2: wp_options のアクティブプラグイン確認
- [ ] I-3: wp_options へのスクリプト注入チェック
- [ ] I-4: wp_users の不審アカウント確認
- [ ] I-5: wp_usermeta の管理者権限確認
- [ ] I-6: wp_posts へのスクリプト注入チェック
- [ ] I-7: DB 全体への包括的パターンスキャン

## 発見した問題

（監査中に発見した問題をここに追記）

## 最終サマリー

（全フェーズ完了後に記入）
```

---

## Phase A: WordPress ルート直下ファイル検査

### A-1: ルート直下 PHP ファイルの一覧

```bash
WP_ROOT="$(realpath "$(pwd)/../../../" 2>/dev/null)"

echo "=== WP ルート直下の PHP ファイル一覧 ==="
find "$WP_ROOT" -maxdepth 1 -name "*.php" -ls | sort -k11

echo "=== 各ファイルの行数 ==="
wc -l "$WP_ROOT"/*.php 2>/dev/null
```

WP 標準ファイル以外（`index.php`, `wp-activate.php`, `wp-blog-header.php`, `wp-comments-post.php`, `wp-config.php`, `wp-config-sample.php`, `wp-cron.php`, `wp-links-opml.php`, `wp-load.php`, `wp-login.php`, `wp-mail.php`, `wp-settings.php`, `wp-signup.php`, `wp-trackback.php`, `xmlrpc.php`）が存在する場合は **不審ファイル** として記録。

### A-2: wp-config.php の安全性確認

Read ツールで `wp-config.php` を読み込み、以下を確認する：

- `DB_NAME`, `DB_USER`, `DB_PASSWORD`, `DB_HOST` — 外部 DB 接続がないか
- `AUTH_KEY` 等シークレットキー — デフォルト値 `put your unique phrase here` のままでないか
- `WP_DEBUG` — `true` になっていないか
- `$table_prefix` — `wp_` のままでないか（セキュリティ上推奨しない）
- 末尾や先頭に `eval`・`base64_decode`・`str_rot13` 等がないか

### A-3〜A-7: コアファイル末尾・行数確認

```bash
WP_ROOT="$(realpath "$(pwd)/../../../" 2>/dev/null)"

echo "=== wp-blog-header.php（標準: 2行程度） ==="
cat "$WP_ROOT/wp-blog-header.php"

echo "=== index.php（標準: 1行 require） ==="
cat "$WP_ROOT/index.php"

echo "=== wp-login.php 行数（標準: 800〜1100行） ==="
wc -l "$WP_ROOT/wp-login.php"

echo "=== wp-settings.php 末尾20行 ==="
tail -20 "$WP_ROOT/wp-settings.php"

echo "=== wp-cron.php 末尾10行 ==="
tail -10 "$WP_ROOT/wp-cron.php"

echo "=== wp-config.php に危険パターン ==="
grep -n "eval\|base64_decode\|str_rot13\|system\|shell_exec\|assert\|preg_replace.*\/e\|file_get_contents.*http" \
  "$WP_ROOT/wp-config.php" 2>/dev/null
```

---

## Phase B: テーマ検査（全テーマ）

### B-1: インストール済みテーマ一覧

```bash
WP_ROOT="$(realpath "$(pwd)/../../../" 2>/dev/null)"
echo "=== インストール済みテーマ ==="
ls -la "$WP_ROOT/wp-content/themes/"
```

### B-2: 全テーマの危険パターンスキャン

```bash
WP_ROOT="$(realpath "$(pwd)/../../../" 2>/dev/null)"
THEMES="$WP_ROOT/wp-content/themes"

echo "=== [テーマ] eval() ==="
grep -rn "eval\s*(" "$THEMES" --include="*.php" 2>/dev/null | grep -v "//.*eval"

echo "=== [テーマ] base64_decode ==="
grep -rn "base64_decode" "$THEMES" --include="*.php" 2>/dev/null | grep -v "//.*base64"

echo "=== [テーマ] システムコマンド ==="
grep -rn -E "(system|shell_exec|exec|passthru|popen)\s*\(" "$THEMES" --include="*.php" 2>/dev/null | grep -v "//.*\(system\|shell\|exec\)"

echo "=== [テーマ] str_rot13 ==="
grep -rn "str_rot13" "$THEMES" --include="*.php" 2>/dev/null

echo "=== [テーマ] file_get_contents + http ==="
grep -rn "file_get_contents" "$THEMES" --include="*.php" 2>/dev/null | grep -i "http\|ftp"

echo "=== [テーマ] eval + \$_REQUEST/GET/POST ==="
grep -rn -E 'eval\s*\(\s*\$_(REQUEST|GET|POST|COOKIE|SERVER)' "$THEMES" --include="*.php" 2>/dev/null

echo "=== [テーマ] GIF89 偽装 ==="
grep -rn "GIF89" "$THEMES" --include="*.php" 2>/dev/null

echo "=== [テーマ] hex2bin / pack ==="
grep -rn -E "(hex2bin|pack\s*\(\s*['\"]H)" "$THEMES" --include="*.php" 2>/dev/null

echo "=== [テーマ] バイナリデータを含む PHP ==="
find "$THEMES" -name "*.php" | while read f; do
  file "$f" 2>/dev/null | grep -qvE "ASCII|UTF-8|empty" && echo "バイナリ: $f"
done

echo "=== [テーマ] 50KB超の PHP ファイル ==="
find "$THEMES" -name "*.php" -size +50k -ls 2>/dev/null
```

### B-3: テーマ内のランダムファイル名チェック（再掲）

```bash
WP_ROOT="$(realpath "$(pwd)/../../../" 2>/dev/null)"
find "$WP_ROOT/wp-content/themes" -name "*.php" | while read f; do
  basename "$f" | grep -qE '^[a-z0-9]{4,12}\.php$' && \
  basename "$f" | grep -qvE '^(index|header|footer|functions|style|page|single|archive|template|sidebar|search|404|comments|content|loop|widget|class|init|setup|about|readme|license|sample|front|home|blog)\.php$' && \
  echo "要確認: $f"
done
```

要確認と出たファイルは Read ツールで内容を確認する。

### B-4: テーマ .htaccess の確認

```bash
WP_ROOT="$(realpath "$(pwd)/../../../" 2>/dev/null)"
find "$WP_ROOT/wp-content/themes" -name ".htaccess" -ls
```

---

## Phase C: プラグイン検査（全プラグイン）

### C-1: インストール済みプラグイン一覧

```bash
WP_ROOT="$(realpath "$(pwd)/../../../" 2>/dev/null)"
echo "=== インストール済みプラグイン ==="
ls -la "$WP_ROOT/wp-content/plugins/"
```

### C-2: 全プラグインの危険パターンスキャン

```bash
WP_ROOT="$(realpath "$(pwd)/../../../" 2>/dev/null)"
PLUGINS="$WP_ROOT/wp-content/plugins"

echo "=== [プラグイン] eval() ==="
grep -rn "eval\s*(" "$PLUGINS" --include="*.php" 2>/dev/null | grep -v "//.*eval" | grep -v "vendor/"

echo "=== [プラグイン] base64_decode ==="
grep -rn "base64_decode" "$PLUGINS" --include="*.php" 2>/dev/null | grep -v "//.*base64" | grep -v "vendor/"

echo "=== [プラグイン] システムコマンド ==="
grep -rn -E "(system|shell_exec|exec|passthru|popen)\s*\(" "$PLUGINS" --include="*.php" 2>/dev/null | grep -v "vendor/"

echo "=== [プラグイン] str_rot13 ==="
grep -rn "str_rot13" "$PLUGINS" --include="*.php" 2>/dev/null | grep -v "vendor/"

echo "=== [プラグイン] file_get_contents + http ==="
grep -rn "file_get_contents" "$PLUGINS" --include="*.php" 2>/dev/null | grep -i "http\|ftp" | grep -v "vendor/"

echo "=== [プラグイン] GIF89 偽装 ==="
grep -rn "GIF89" "$PLUGINS" --include="*.php" 2>/dev/null

echo "=== [プラグイン] バイナリデータを含む PHP ==="
find "$PLUGINS" -name "*.php" -not -path "*/vendor/*" | while read f; do
  file "$f" 2>/dev/null | grep -qvE "ASCII|UTF-8|empty" && echo "バイナリ: $f"
done

echo "=== [プラグイン] 50KB超の PHP ==="
find "$PLUGINS" -name "*.php" -not -path "*/vendor/*" -size +50k -ls 2>/dev/null
```

### C-3: プラグイン内のランダムファイル名チェック

```bash
WP_ROOT="$(realpath "$(pwd)/../../../" 2>/dev/null)"
find "$WP_ROOT/wp-content/plugins" -name "*.php" -not -path "*/vendor/*" | while read f; do
  basename "$f" | grep -qE '^[a-z0-9]{4,10}\.php$' && \
  basename "$f" | grep -qvE '^(index|class|core|init|load|ajax|api|admin|widget|plugin|setup|install|update|upgrade|cron|mail|feed|functions|template|options|settings|utils|helpers|loader|public|includes|vendor|autoload|bootstrap|config|cache|log|auth|user|post|page|menu|meta|hook|filter|action|shortcode|block|field|form|media|upload|asset|script|style|view|model|controller|router|request|response|session|cookie|nonce|sanitize|validate|format|i18n|locale|date|time|number|string|array|object|file|http|url|image|email|sms|payment|shipping|tax|order|product|customer|report|export|import|sync|queue|task|job|worker|event|notification|webhook|oauth|jwt|rest|graphql|database|query|schema|migration|seed|factory|test|spec|mock|stub|fixture|helper|trait|interface|abstract|enum|constant|exception|error|debug|log|monitor|health|status|version|license|readme|about|changelog|uninstall|activate|deactivate)\.php$' && \
  echo "要確認: $f"
done
```

---

## Phase D: wp-content 直下・uploads 検査

### D-1: wp-content 直下の不審ファイル

```bash
WP_ROOT="$(realpath "$(pwd)/../../../" 2>/dev/null)"

echo "=== wp-content 直下のファイル ==="
ls -la "$WP_ROOT/wp-content/"

echo "=== wp-content 直下の PHP ファイル内容確認 ==="
find "$WP_ROOT/wp-content" -maxdepth 1 -name "*.php" -ls
```

Read ツールで `wp-content/index.php` を確認する（標準は `<?php // Silence is golden.` の1行のみ）。
他に PHP ファイルがあれば内容を確認する。

### D-2〜D-4: uploads の詳細検査

uploads の詳細検査は `/uploads-scan` スキルで実施する。

```bash
WP_ROOT="$(realpath "$(pwd)/../../../" 2>/dev/null)"
UPLOADS="$WP_ROOT/wp-content/uploads"
```

**`/uploads-scan "$UPLOADS"` を実行すること。**

簡易確認のみ行う場合：

```bash
WP_ROOT="$(realpath "$(pwd)/../../../" 2>/dev/null)"

echo "=== uploads 内の PHP ファイル（0件が正常） ==="
find "$WP_ROOT/wp-content/uploads" -type f \( \
  -iname "*.php" -o -iname "*.phtml" -o -iname "*.php5" -o \
  -iname "*.php7" -o -iname "*.phar" -o -iname "*.sh" -o \
  -iname "*.py" -o -iname "*.pl" \
\) -ls 2>/dev/null

echo "=== uploads 内の .htaccess ==="
find "$WP_ROOT/wp-content/uploads" -name ".htaccess" -ls 2>/dev/null

echo "=== uploads 内の最近変更された非画像ファイル ==="
find "$WP_ROOT/wp-content/uploads" -mtime -30 -type f \
  -not \( -iname "*.jpg" -o -iname "*.jpeg" -o -iname "*.png" -o \
  -iname "*.gif" -o -iname "*.webp" -o -iname "*.pdf" -o \
  -iname "*.mp4" -o -iname "*.svg" -o -iname "*.ico" \) \
  -ls 2>/dev/null
```

---

## Phase E: WordPress コア整合性確認

コアは全件確認ではなく、**混入しやすい箇所と危険パターンに絞る**。

### E-1: wp-includes 内の危険パターン

```bash
WP_ROOT="$(realpath "$(pwd)/../../../" 2>/dev/null)"

echo "=== wp-includes 内の eval（正規コードは eval を使わない） ==="
grep -rn "eval\s*(" "$WP_ROOT/wp-includes" --include="*.php" 2>/dev/null | \
  grep -v "//.*eval" | grep -v "preg_replace.*\/e.*deprecated" | head -20

echo "=== wp-includes 内の 50KB 超 PHP ==="
find "$WP_ROOT/wp-includes" -name "*.php" -size +200k -ls 2>/dev/null

echo "=== wp-includes 内の最近変更ファイル（30日以内） ==="
find "$WP_ROOT/wp-includes" -mtime -30 -name "*.php" -ls 2>/dev/null
```

### E-2: wp-admin 内の危険パターン

```bash
WP_ROOT="$(realpath "$(pwd)/../../../" 2>/dev/null)"

echo "=== wp-admin 内の eval ==="
grep -rn "eval\s*(" "$WP_ROOT/wp-admin" --include="*.php" 2>/dev/null | \
  grep -v "//.*eval" | head -20

echo "=== wp-admin 内の最近変更ファイル（30日以内） ==="
find "$WP_ROOT/wp-admin" -mtime -30 -name "*.php" -ls 2>/dev/null
```

### E-3: コアファイルのベーシックチェック

```bash
WP_ROOT="$(realpath "$(pwd)/../../../" 2>/dev/null)"

echo "=== wp-load.php 行数（標準: 100行前後） ==="
wc -l "$WP_ROOT/wp-load.php"

echo "=== wp-mail.php 末尾 ==="
tail -5 "$WP_ROOT/wp-mail.php"

echo "=== wp-signup.php 末尾 ==="
tail -5 "$WP_ROOT/wp-signup.php"
```

---

## Phase F: .htaccess 全箇所確認

```bash
WP_ROOT="$(realpath "$(pwd)/../../../" 2>/dev/null)"

echo "=== サイト内の全 .htaccess ファイル ==="
find "$WP_ROOT" -name ".htaccess" -ls 2>/dev/null
```

Read ツールで各 .htaccess を読み込み、以下を確認する：

| 確認場所 | 確認内容 |
|---------|---------|
| WP ルート | WordPress リライトルール・wp-config.php 保護・xmlrpc 無効化・ディレクトリ一覧禁止・セキュリティヘッダー・HTTPS リダイレクト |
| テーマ | PHP 直接実行禁止 |
| uploads | PHP 実行禁止（最重要） |
| その他 | 不審な設定（外部リダイレクト・RewriteRule で外部へ飛ばすなど）がないか |

不審な設定例（マルウェアが挿入するケース）:
```apache
# 外部への強制リダイレクト
RewriteCond %{HTTP_USER_AGENT} (Googlebot|bingbot)
RewriteRule .* http://malicious-site.com/ [R=301,L]
```

---

## Phase G: ファイルパーミッション

### G-1〜G-4: パーミッション一括確認

```bash
WP_ROOT="$(realpath "$(pwd)/../../../" 2>/dev/null)"

echo "=== 他者書き込み可能（最危険） ==="
find "$WP_ROOT" \( -perm -o+w \) -not -path "*/.git/*" -ls 2>/dev/null | grep -v "vendor/"

echo "=== グループ書き込み可能（要確認） ==="
find "$WP_ROOT" \( -perm -g+w \) -not -path "*/.git/*" \
  -not -path "*/wp-content/uploads/*" \
  -ls 2>/dev/null | grep -v "vendor/" | head -20

echo "=== PHP ファイルに実行ビット ==="
find "$WP_ROOT" -name "*.php" -perm /111 -ls 2>/dev/null

echo "=== wp-config.php パーミッション（推奨: 600 または 640） ==="
ls -la "$WP_ROOT/wp-config.php"

echo "=== .htaccess パーミッション（推奨: 644） ==="
find "$WP_ROOT" -name ".htaccess" -ls 2>/dev/null
```

---

## Phase H: 設定・認証情報確認

### H-1〜H-4: wp-config.php 詳細確認

Read ツールで `wp-config.php` を開き、以下のチェックリストを確認する：

```
[ ] シークレットキーが 'put your unique phrase here' でない
[ ] WP_DEBUG が false（または未定義）
[ ] $table_prefix が 'wp_' 以外（変更推奨だが既存サイトは慎重に）
[ ] DISALLOW_FILE_EDIT が true（管理画面からのテーマ/プラグイン編集禁止）
[ ] DISALLOW_FILE_MODS が true（更新機能の無効化、必要に応じて）
[ ] DB_HOST が localhost（外部DB接続でないこと）
[ ] FORCE_SSL_ADMIN が true（SSL環境の場合）
```

### H-4: xmlrpc.php アクセス確認

.htaccess で xmlrpc.php がブロックされているか確認（Phase F で確認済みの場合はスキップ）。

### H-5: ユーザー名漏洩対策の確認

```bash
echo "=== /wp-json/wp/v2/users 無効化の確認 ==="
grep -n "rest_endpoints\|wp/v2/users" "$(pwd)/functions.php" 2>/dev/null || \
  echo "未設定 — ユーザー名列挙が可能な状態"

echo "=== 著者アーカイブ無効化の確認 ==="
grep -n "author_rewrite_rules\|is_author" "$(pwd)/functions.php" 2>/dev/null || \
  echo "未設定 — /?author=1 でユーザー名が漏洩する可能性あり"
```

未設定の場合は `/htaccess-security` スキルの functions.php セクションを参照して追記を案内する。

### H-6: WAF の導入確認

以下を確認してユーザーに報告する：

```bash
echo "=== Wordfence プラグインの確認 ==="
find "$(pwd)/../../../wp-content/plugins" -maxdepth 1 -name "wordfence" -type d 2>/dev/null || \
  echo "Wordfence 未導入"

echo "=== Sucuri プラグインの確認 ==="
find "$(pwd)/../../../wp-content/plugins" -maxdepth 1 -name "sucuri-scanner" -type d 2>/dev/null || \
  echo "Sucuri 未導入"
```

WAF が未導入の場合は以下を案内する：
- **Cloudflare**（推奨）: DNS を Cloudflare 経由にするだけで WAF・DDoS対策・CDN が有効になる。無料プランあり。
- **Wordfence**: WordPress プラグイン型。ファイル改ざん検知・ログイン保護も兼ねる。無料プランあり。

---

## Phase I: データベース検査

**.sql ファイルがある場合のみ実施。** サーバー移行前の DB 検査として特に重要。

ユーザーに「検査対象の .sql ファイルのパスを教えてください」と確認してから開始する。
パスが提供されたら `/db-scan [パス]` を実行する。

`/db-scan` が実施済みの場合はその結果を参照してチェック項目を更新する。

チェック内容（詳細は `/db-scan` スキル参照）：
- wp_options: siteurl / home の改ざん・スクリプト注入
- wp_users: 不審な管理者アカウント
- wp_usermeta: 権限昇格の痕跡
- wp_posts: スクリプト・eval 注入
- DB 全体: base64 / eval / 外部 URL パターン

---

## 監査完了後: 進捗ファイルの更新とレポート出力

全フェーズ完了後、`.security-audit-progress.md` を更新して以下の総合レポートを出力する：

```markdown
## セキュリティ監査 総合レポート

**監査実施日時:** [日時]
**対象サイト:** [WP ルートパス]
**WordPress バージョン:** [バージョン]

### 危険度別 発見事項

#### 🔴 即座に対応が必要
（発見した問題をここに列挙）

#### 🟡 早急に対応が推奨される
（発見した問題をここに列挙）

#### 🟢 改善推奨
（発見した問題をここに列挙）

### フェーズ別 結果サマリー

| フェーズ | 結果 | 詳細 |
|---------|------|------|
| Phase 0: 棚卸し | ✅/⚠️/❌ | |
| Phase A: WP ルート | ✅/⚠️/❌ | |
| Phase B: テーマ | ✅/⚠️/❌ | |
| Phase C: プラグイン | ✅/⚠️/❌ | |
| Phase D: uploads | ✅/⚠️/❌ | |
| Phase E: コア整合性 | ✅/⚠️/❌ | |
| Phase F: .htaccess | ✅/⚠️/❌ | |
| Phase G: パーミッション | ✅/⚠️/❌ | |
| Phase H: 設定確認 | ✅/⚠️/❌ | |
| Phase I: データベース | ✅/⚠️/❌/⏭️ スキップ（.sqlなし） | |

### 次のアクション

1. （優先順に対応内容を記載）

### 使用するコマンド

- `/malware-clean [ファイル]` — マルウェアの検疫・削除
- `/htaccess-security` — .htaccess の強化
- `/db-scan [.sqlファイル]` — DB の詳細検査
- `site-hardener エージェント` — パーミッション・wp-config の修正
```
