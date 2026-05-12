---
name: uploads-scan
description: WordPress の uploads ディレクトリを専門的に検査するスキル。PHP混入・実行ファイル・画像に偽装したマルウェア・SVGスクリプト注入・不審なHTMLファイルを網羅的にチェックする。サーバー移行前の uploads 検査として使用する。
argument-hint: [uploadsディレクトリのパス（例: ~/Downloads/uploads）]
---

あなたは WordPress セキュリティの専門家です。
指定された uploads ディレクトリを徹底的に検査し、マルウェア・不審なファイルをすべて検出して日本語でレポートしてください。

**対象ディレクトリ：** $ARGUMENTS

> uploads ディレクトリに PHP ファイルは **1件も存在してはいけません。** 1件でも見つかった場合は即座に危険と判断してください。

---

## 事前確認

```bash
TARGET="$ARGUMENTS"
echo "=== 対象ディレクトリの確認 ==="
ls -la "$TARGET"

echo "=== 総ファイル数 ==="
find "$TARGET" -type f | wc -l

echo "=== ディレクトリ構成 ==="
find "$TARGET" -type d | head -30

echo "=== 拡張子別ファイル数 ==="
find "$TARGET" -type f | sed 's/.*\.//' | sort | uniq -c | sort -rn | head -30
```

---

## Step 1: 実行可能ファイルの検出（最重要）

### 1-1: PHP ファイルの完全検出

```bash
TARGET="$ARGUMENTS"

echo "=== PHP 拡張子ファイル（0件が正常） ==="
find "$TARGET" -type f \( \
  -iname "*.php" -o \
  -iname "*.php3" -o \
  -iname "*.php4" -o \
  -iname "*.php5" -o \
  -iname "*.php7" -o \
  -iname "*.php8" -o \
  -iname "*.phtml" -o \
  -iname "*.phar" \
\) -ls 2>/dev/null
```

**1件でも検出された場合: 即座に `/malware-clean` で検疫すること。**

### 1-2: その他の実行スクリプト

```bash
TARGET="$ARGUMENTS"

echo "=== シェル・スクリプトファイル ==="
find "$TARGET" -type f \( \
  -iname "*.sh" -o \
  -iname "*.py" -o \
  -iname "*.pl" -o \
  -iname "*.rb" -o \
  -iname "*.cgi" -o \
  -iname "*.asp" -o \
  -iname "*.aspx" -o \
  -iname "*.exe" -o \
  -iname "*.bat" \
\) -ls 2>/dev/null
```

---

## Step 2: 拡張子偽装ファイルの検出（高度な手口）

画像・PDFに見せかけて PHP コードを仕込む手口を検出する。

### 2-1: 拡張子と実際のファイルタイプの不一致

```bash
TARGET="$ARGUMENTS"

echo "=== 拡張子が画像だが実際には異なるファイル ==="
find "$TARGET" -type f \( \
  -iname "*.jpg" -o -iname "*.jpeg" -o \
  -iname "*.png" -o -iname "*.gif" -o \
  -iname "*.webp" -o -iname "*.bmp" \
\) | while read f; do
  actual=$(file "$f" 2>/dev/null | cut -d: -f2)
  echo "$actual" | grep -qiE "JPEG|PNG|GIF|WebP|bitmap|BMP" || \
    echo "不一致: $f → $actual"
done
```

### 2-2: 画像ファイル内の PHP コード混入

```bash
TARGET="$ARGUMENTS"

echo "=== 画像ファイル内の PHP タグ ==="
grep -rl "<?php\|<?=" "$TARGET" --include="*.jpg" --include="*.jpeg" \
  --include="*.png" --include="*.gif" --include="*.webp" 2>/dev/null

echo "=== 画像ファイル内の eval ==="
grep -rl "eval\s*(" "$TARGET" --include="*.jpg" --include="*.jpeg" \
  --include="*.png" --include="*.gif" --include="*.webp" 2>/dev/null

echo "=== 画像ファイル内の base64_decode ==="
grep -rl "base64_decode" "$TARGET" --include="*.jpg" --include="*.jpeg" \
  --include="*.png" --include="*.gif" --include="*.webp" 2>/dev/null
```

### 2-3: PDF・ドキュメントファイルの偽装確認

```bash
TARGET="$ARGUMENTS"

echo "=== PDF 拡張子だが実際には異なるファイル ==="
find "$TARGET" -iname "*.pdf" | while read f; do
  actual=$(file "$f" 2>/dev/null | cut -d: -f2)
  echo "$actual" | grep -qi "PDF" || echo "不一致: $f → $actual"
done
```

---

## Step 3: SVG・HTML ファイルの検査

SVG と HTML は画像・コンテンツとして uploads に置かれることがあるが、スクリプトを含められる危険なフォーマット。

### 3-1: SVG ファイルのスクリプト検査

```bash
TARGET="$ARGUMENTS"

echo "=== SVG ファイル一覧 ==="
find "$TARGET" -iname "*.svg" -ls 2>/dev/null

echo "=== SVG 内の <script> タグ ==="
grep -rn "<script" "$TARGET" --include="*.svg" 2>/dev/null

echo "=== SVG 内の JavaScript イベントハンドラ ==="
grep -rn -iE "onload=|onerror=|onclick=|onmouseover=|onfocus=" \
  "$TARGET" --include="*.svg" 2>/dev/null

echo "=== SVG 内の外部リソース参照 ==="
grep -rn -iE "href=['\"]https?://" "$TARGET" --include="*.svg" 2>/dev/null | \
  grep -v "xmlns"
```

### 3-2: HTML・HTM ファイルの検査

```bash
TARGET="$ARGUMENTS"

echo "=== HTML ファイル一覧（uploads に HTML は不審） ==="
find "$TARGET" -type f \( -iname "*.html" -o -iname "*.htm" \) -ls 2>/dev/null

echo "=== HTML 内の <script> タグ ==="
grep -rn "<script" "$TARGET" --include="*.html" --include="*.htm" 2>/dev/null

echo "=== HTML 内の外部スクリプト読み込み ==="
grep -rn -iE "src=['\"]https?://" "$TARGET" \
  --include="*.html" --include="*.htm" 2>/dev/null
```

---

## Step 4: JavaScript ファイルの検査

```bash
TARGET="$ARGUMENTS"

echo "=== JS ファイル一覧（uploads に JS は不審） ==="
find "$TARGET" -iname "*.js" -ls 2>/dev/null

echo "=== JS 内の eval / document.write ==="
grep -rn -iE "eval\s*\(|document\.write\s*\(" "$TARGET" --include="*.js" 2>/dev/null

echo "=== JS 内の難読化パターン ==="
grep -rn -iE "String\.fromCharCode|atob\s*\(|unescape\s*\(" \
  "$TARGET" --include="*.js" 2>/dev/null
```

---

## Step 5: ファイル名の検査

### 5-1: 不審なファイル名パターン

```bash
TARGET="$ARGUMENTS"

echo "=== ランダムな英数字ファイル名（マルウェアの特徴） ==="
find "$TARGET" -type f | while read f; do
  base=$(basename "$f")
  # 拡張子を除いたファイル名が 5〜12 文字のランダム英数字
  name="${base%.*}"
  echo "$name" | grep -qE '^[a-z0-9]{5,12}$' && echo "要確認: $f"
done

echo "=== ダブル拡張子ファイル（例: image.php.jpg） ==="
find "$TARGET" -type f -name "*.php.*" -o -name "*.phtml.*" 2>/dev/null | head -20

echo "=== スペース・特殊文字を含むファイル名 ==="
find "$TARGET" -type f -name "* *" -o -name "*[;&|<>]*" 2>/dev/null | head -20

echo "=== ドットファイル（隠しファイル） ==="
find "$TARGET" -name ".*" -type f -ls 2>/dev/null
```

### 5-2: .htaccess ファイルの確認

```bash
TARGET="$ARGUMENTS"

echo "=== uploads 内の .htaccess ==="
find "$TARGET" -name ".htaccess" -ls 2>/dev/null
```

.htaccess が存在する場合は Read ツールで内容を確認し、不審なリダイレクトや PHP 実行許可設定がないか確認する。
正常な uploads の .htaccess は PHP 実行禁止設定のみであるべき。

---

## Step 6: ファイルサイズの検査

```bash
TARGET="$ARGUMENTS"

echo "=== 異常に大きいファイル（画像として 10MB 超は要確認） ==="
find "$TARGET" -type f -size +10M -ls 2>/dev/null

echo "=== 0バイトファイル（不審） ==="
find "$TARGET" -type f -empty -ls 2>/dev/null

echo "=== 異常に大きい画像ファイル（5MB 超） ==="
find "$TARGET" -type f \( \
  -iname "*.jpg" -o -iname "*.jpeg" -o \
  -iname "*.png" -o -iname "*.gif" -o -iname "*.webp" \
\) -size +5M -ls 2>/dev/null
```

---

## Step 7: 最近変更されたファイルの確認

```bash
TARGET="$ARGUMENTS"

echo "=== 30日以内に変更されたファイル（全種類） ==="
find "$TARGET" -mtime -30 -type f -ls 2>/dev/null | sort -k8,9

echo "=== 30日以内に変更された非画像ファイル（最重要） ==="
find "$TARGET" -mtime -30 -type f \
  -not \( \
    -iname "*.jpg" -o -iname "*.jpeg" -o -iname "*.png" -o \
    -iname "*.gif" -o -iname "*.webp" -o -iname "*.pdf" -o \
    -iname "*.mp4" -o -iname "*.mov" -o -iname "*.mp3" -o \
    -iname "*.svg" -o -iname "*.ico" \
  \) \
  -ls 2>/dev/null | sort -k8,9
```

---

## Step 8: バイナリ・暗号化ファイルの検出

```bash
TARGET="$ARGUMENTS"

echo "=== バイナリデータを含む非画像ファイル ==="
find "$TARGET" -type f \
  -not \( \
    -iname "*.jpg" -o -iname "*.jpeg" -o -iname "*.png" -o \
    -iname "*.gif" -o -iname "*.webp" -o -iname "*.bmp" -o \
    -iname "*.ico" -o -iname "*.pdf" -o -iname "*.mp4" -o \
    -iname "*.mov" -o -iname "*.mp3" -o -iname "*.woff" -o \
    -iname "*.woff2" -o -iname "*.ttf" -o -iname "*.eot" \
  \) | while read f; do
  file "$f" 2>/dev/null | grep -qvE "ASCII|UTF-8|empty|script|text" && \
    echo "バイナリ: $f → $(file "$f" | cut -d: -f2)"
done | head -20
```

---

## Step 9: シンボリックリンクの確認

```bash
TARGET="$ARGUMENTS"

echo "=== シンボリックリンク（不審な場合がある） ==="
find "$TARGET" -type l -ls 2>/dev/null
```

---

## 疑いファイルの内容確認

Step 1〜9 で疑いのあるファイルが見つかった場合：
- Read ツールでファイルの内容を読み込む
- 深掘り分析が必要な場合は `malware-investigator` エージェントに依頼する

---

## レポート形式

```markdown
## uploads スキャンレポート

**対象ディレクトリ:** [パス]
**スキャン日時:** [日時]
**総ファイル数:** [件数]

### 判定サマリー

| 判定 | 件数 |
|------|------|
| 🔴 即座に削除（PHP・実行ファイル） | X 件 |
| 🟡 要確認（偽装・不審パターン） | X 件 |
| ✅ 問題なし | X 件 |

### 🔴 即座に削除が必要なファイル

| ファイル | 理由 |
|---------|------|
| uploads/2024/03/shell.php | PHP ファイル混入 |

### 🟡 要確認ファイル

| ファイル | 理由 |
|---------|------|
| uploads/2024/01/image.jpg | 実際のファイルタイプが JPEG でない |
| uploads/2024/02/banner.svg | <script> タグを含む |

### ✅ 問題なし

（問題のなかったファイル種別・件数）

### 次のアクション

（問題があれば）
1. 🔴 のファイルを `/malware-clean` で検疫・削除
2. 🟡 のファイルを個別に確認して判断
3. 問題がなければ移行を続行
```

---

## 移行の可否判断

| 状態 | 判断 |
|------|------|
| PHP・実行ファイルが 0 件、偽装なし | ✅ **移行可能** |
| PHP ファイルが検出された | ❌ **削除してから再検査** |
| 偽装・不審ファイルが検出された | ⚠️ **個別確認後に判断** |
