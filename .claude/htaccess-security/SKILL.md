---
name: htaccess-security
description: .htaccess のセキュリティ対策を現在のファイルに追記・レビューする。不足している設定を指摘し、必要に応じて安全なコードブロックを生成する。WordPress環境（WPルート・テーマ・uploadsの3箇所）を網羅的にチェックする。
argument-hint: [.htaccessのパス または 対象のディレクトリパス（省略時はWPルート・テーマ・uploadsを全て確認）]
---

あなたは Apache サーバーのセキュリティ専門家です。
WordPress サイトの `.htaccess` ファイルを全箇所確認し、不足している設定を特定して追記を提案してください。

**対象：** $ARGUMENTS（省略時は WordPress ルート・テーマ・uploads を全て確認）

## 手順

1. 対象の `.htaccess` を Read ツールで読み込む
   - 引数がなければ以下の3箇所を全て確認する：
     - WP ルート: `../../../.htaccess`
     - テーマ: `./.htaccess`
     - uploads: `../../../wp-content/uploads/.htaccess`
2. 下記チェックリストと照合して現状を評価する
3. 不足項目を特定し、追記すべきコードブロックを生成する
4. ユーザーに確認を取ってから Edit ツールで追記する（uploads に .htaccess がない場合は Write ツールで新規作成）

---

## セキュリティチェックリスト

### 【WPルート .htaccess】

#### 1. WordPress 標準リライト設定（確認のみ）
WordPress 標準のリライトルールが存在し、BEGIN/END マーカーが正常か確認する。

#### 2. wp-config.php のアクセス禁止
```apache
# wp-config.php を保護
<Files wp-config.php>
  <IfModule mod_authz_core.c>
    Require all denied
  </IfModule>
  <IfModule !mod_authz_core.c>
    Order deny,allow
    Deny from all
  </IfModule>
</Files>
```

#### 3. xmlrpc.php の無効化
```apache
# xmlrpc.php を無効化（REST API を使用している場合は不要）
<Files xmlrpc.php>
  <IfModule mod_authz_core.c>
    Require all denied
  </IfModule>
  <IfModule !mod_authz_core.c>
    Order deny,allow
    Deny from all
  </IfModule>
</Files>
```

#### 4. ディレクトリ一覧表示の禁止
```apache
Options -Indexes
```

#### 5. ドットファイル・隠しファイルのアクセス禁止
```apache
<IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteRule (^|/)\. - [F,L]
</IfModule>
```

#### 6. 開発・設定ファイルのアクセス禁止
```apache
<FilesMatch "\.(json|lock|sh|env|sql|log|md|xml|yml|yaml|ini|conf)$">
  <IfModule mod_authz_core.c>
    Require all denied
  </IfModule>
  <IfModule !mod_authz_core.c>
    Order deny,allow
    Deny from all
  </IfModule>
</FilesMatch>
```

#### 7. セキュリティヘッダーの付与
```apache
<IfModule mod_headers.c>
  Header always set X-Frame-Options "SAMEORIGIN"
  Header always set X-Content-Type-Options "nosniff"
  Header always set X-XSS-Protection "1; mode=block"
  Header always set Referrer-Policy "strict-origin-when-cross-origin"
  # CSP はサイト要件に合わせて調整すること
  # Header always set Content-Security-Policy "default-src 'self';"
</IfModule>
```

#### 8. HTTPS へのリダイレクト
```apache
<IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteCond %{HTTPS} off
  RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
</IfModule>
```

#### 9. 不審な HTTP メソッドの禁止
```apache
<LimitExcept GET POST HEAD>
  Order deny,allow
  Deny from all
</LimitExcept>
```

#### 10. PHP エラー表示の無効化（本番環境）
```apache
<IfModule mod_php.c>
  php_flag display_errors Off
  php_flag log_errors On
</IfModule>
```

---

### 【テーマ .htaccess】

現在のテーマ `.htaccess` の内容を確認し、以下のみを対象とする（WP リライトと干渉しないよう注意）：

#### PHP ファイルの直接実行禁止（テーマディレクトリは基本的に PHP を直接呼び出さない）
```apache
<FilesMatch "\.(php|phtml|php3|php4|php5|php7|cgi|pl|sh|py|rb)$">
  <IfModule mod_authz_core.c>
    Require all denied
  </IfModule>
  <IfModule !mod_authz_core.c>
    Order deny,allow
    Deny from all
  </IfModule>
</FilesMatch>
```

> 注意: テーマの PHP ファイルは WordPress が内部でインクルードするため、直接アクセスを禁止しても正常動作する。ただし `functions.php` 等を直接アクセスする仕組みがある場合は除外すること。ユーザーに確認を取る。

---

### 【uploads .htaccess】

uploads ディレクトリは PHP を実行する必要が一切ない。

#### PHP 実行禁止（最重要）
```apache
# アップロードディレクトリでの PHP 実行を禁止
<FilesMatch "\.(php|phtml|php3|php4|php5|php7|cgi|pl|sh|py|rb)$">
  <IfModule mod_authz_core.c>
    Require all denied
  </IfModule>
  <IfModule !mod_authz_core.c>
    Order deny,allow
    Deny from all
  </IfModule>
</FilesMatch>
```

---

### 【functions.php セキュリティ設定】

.htaccess では対応できない WordPress 固有のセキュリティ設定。
テーマの `functions.php` に設定されているか確認し、不足があれば追記を提案する。

#### /wp-json/wp/v2/users の無効化（ユーザー名列挙対策）

このエンドポイントが開いていると誰でもユーザー名一覧を取得でき、ブルートフォース攻撃の前準備に悪用される。

```php
// REST API のユーザー列挙を無効化
add_filter('rest_endpoints', function($endpoints) {
    if (isset($endpoints['/wp/v2/users'])) {
        unset($endpoints['/wp/v2/users']);
    }
    if (isset($endpoints['/wp/v2/users/(?P<id>[\d]+)'])) {
        unset($endpoints['/wp/v2/users/(?P<id>[\d]+)']);
    }
    return $endpoints;
});
```

#### 著者アーカイブページの無効化（ユーザー名漏洩対策）

`/?author=1` などのURLからもユーザー名が漏洩するため、あわせて無効化する。

```php
// 著者アーカイブページを無効化してリダイレクト
add_filter('author_rewrite_rules', '__return_empty_array');
add_action('template_redirect', function() {
    if (is_author()) {
        wp_redirect(home_url(), 301);
        exit;
    }
});
```

> 注意: 著者アーカイブを活用しているサイト（ブログ・メディア系）は無効化前に確認すること。

---

## 出力形式

```
## .htaccess セキュリティレビュー

### 現状評価

#### WP ルート (.htaccess)
| # | 項目 | 状態 | 備考 |
|---|------|------|------|
| 1 | wp-config.php 保護 | ✅ 対応済み / ❌ 未対応 | |
| 2 | xmlrpc.php 無効化 | ✅ / ❌ | |
| 3 | ディレクトリ一覧禁止 | ✅ / ❌ | |
| 4 | ドットファイル禁止 | ✅ / ❌ | |
| 5 | 設定ファイル禁止 | ✅ / ❌ | |
| 6 | セキュリティヘッダー | ✅ / ❌ | |
| 7 | HTTPS リダイレクト | ✅ / ❌ | |
| 8 | 不審な HTTP メソッド禁止 | ✅ / ❌ | |

#### テーマ (.htaccess)
| # | 項目 | 状態 | 備考 |
|---|------|------|------|
| 1 | PHP 直接実行禁止 | ✅ / ❌ | |

#### uploads (.htaccess)
| # | 項目 | 状態 | 備考 |
|---|------|------|------|
| 1 | PHP 実行禁止 | ✅ / ❌ | ファイルなしの場合は新規作成 |

#### functions.php
| # | 項目 | 状態 | 備考 |
|---|------|------|------|
| 1 | /wp-json/wp/v2/users 無効化 | ✅ / ❌ | |
| 2 | 著者アーカイブ無効化 | ✅ / ❌ | 著者ページを使用していない場合 |

### 追記が必要なブロック

（不足している設定のコードブロックを箇所ごとに列挙）

### 注意事項

（サイト固有の注意点や、設定前に確認すべき事項）
```

追記内容をユーザーに確認してから、Edit / Write ツールで各 `.htaccess` に反映してください。
