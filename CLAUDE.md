# CLAUDE.md — WordPress セキュリティワークスペース

## プロジェクト概要

- **サイト:** （プロジェクト導入時に記入）
- **CMS:** WordPress
- **テーマルート:** `wp-content/themes/[テーマ名]/`
- **WP ルート:** テーマルートから `../../../` 上が WordPress ルート
- **状態:** （例: マルウェア感染済み — クリーンアップ作業中）

## 既知のマルウェア

スキャン後に記入する。初期状態は空欄。

| ファイル | 種別 | 脅威レベル |
|---------|------|-----------|
| （`/malware-scan` 実行後に追記） | | |

---

## 自動検査プロトコル（最重要）

ユーザーのメッセージに以下のキーワードが含まれる場合、**説明や確認なしに直ちに `/security-audit` を開始する**こと。

**トリガーキーワード:**
`検査` / `検索` / `調査` / `調べて` / `確認して` / `チェック` / `スキャン` / `ウイルス` / `マルウェア` / `脆弱性` / `感染` / `不正` / `セキュリティ`

**自動起動ルール:**
1. キーワードを検出したら、まず `.security-audit-progress.md` が存在するか確認する
2. **存在する場合:** 未完了（`[ ]` のまま）の項目を一覧表示して「前回の続きから再開しますか？それとも最初から全て実行しますか？」と聞く
3. **存在しない場合:** `/security-audit` を最初から実行する（ユーザーへの事前確認は不要）
4. 各フェーズ完了のたびに `.security-audit-progress.md` を更新して進捗を保存する
5. 全フェーズ完了後に総合レポートを出力する

**未検査項目の通知ルール:**
- 会話の中で「まだチェックしていない項目はありますか？」「漏れはありますか？」と聞かれた場合、`.security-audit-progress.md` を読み込んで `[ ]` のまま残っている項目を全て列挙して報告する
- `.security-audit-progress.md` が存在しない場合は「まだ監査が実施されていません。`/security-audit` を開始しますか？」と答える

---

## クリーンアップワークフロー

```
/security-audit（総合監査・ファイル検査）
     ↓ DB ファイル（.sql）がある場合
/db-scan [ファイル]（DB内のマルウェア・不審アカウント検査）
     ↓ uploads を移行する場合
/uploads-scan [ディレクトリ]（PHP混入・偽装・スクリプト注入検査）
     ↓ 問題が見つかった場合
/malware-clean [ファイル]（検疫・削除）
     ↓
/htaccess-security（.htaccessの強化）
     ↓
site-hardener エージェント（パーミッション・設定の堅牢化）
```

個別スキルの直接実行も可能：
```
/malware-scan      → 危険パターンのスキャンのみ
/malware-clean     → 検疫・削除のみ
/db-scan           → DB の検査のみ
/uploads-scan      → uploads の検査のみ
/htaccess-security → .htaccess レビューのみ
```

---

## Claude への行動ルール

### 必須ルール（例外なし）

1. **検疫優先:** マルウェアファイルを削除する前に必ず `.quarantine/YYYY-MM-DD/` へ移動すること。直接削除は禁止。
2. **WordPress コアは変更しない:** `wp-includes/`、`wp-admin/`、`wp-*.php`（ルート直下）は読み取りのみ。疑わしい場合はユーザーに報告して判断を仰ぐ。
3. **疑いコードを実行しない:** マルウェアと判定されたコードは `eval`・`include`・`require` で実行しない。内容確認は Read ツールのみ。
4. **確認してから実行:** ファイル移動・削除・`.htaccess` 書き換えは必ずユーザーに確認を取ってから行うこと。
5. **ログを残す:** クリーンアップ操作はすべて `.quarantine/cleanup-log.md` に記録すること。
6. **進捗を保存する:** `/security-audit` 実行中は各フェーズ完了ごとに `.security-audit-progress.md` を更新すること。途中で止まっても再開できるようにする。

### 推奨行動

- スキャン結果に疑いファイルが多い場合は `malware-investigator` エージェントに深掘り分析を依頼する。
- クリーンアップ完了後は `site-hardener` エージェントでパーミッション・.htaccess・wp-config を確認する。
- `/security-audit` と `/malware-scan` は WordPress ルート全体（`../../../`）を対象にすること。テーマだけでは不十分。

---

## ディレクトリ構成（重要箇所）

```
[WP ルート]/
├── .htaccess                  ← WP 標準リライトルール（要セキュリティ強化）
├── wp-config.php              ← DB 接続情報（絶対に外部公開禁止）
├── wp-admin/
├── wp-includes/
└── wp-content/
    ├── themes/[テーマ名]/     ← 作業ディレクトリ（ここが Claude の CWD）
    │   ├── CLAUDE.md          ← このファイル
    │   ├── .claude/
    │   │   ├── settings.json
    │   │   ├── security-audit/SKILL.md      (/security-audit) ← 総合監査
    │   │   ├── malware-scan/SKILL.md        (/malware-scan)
    │   │   ├── malware-clean/SKILL.md       (/malware-clean)
    │   │   ├── db-scan/SKILL.md             (/db-scan)
    │   │   ├── uploads-scan/SKILL.md        (/uploads-scan)
    │   │   ├── htaccess-security/SKILL.md   (/htaccess-security)
    │   │   └── agents/
    │   │       ├── malware-investigator.md  ← 深掘り分析
    │   │       └── site-hardener.md         ← 堅牢化
    │   ├── .quarantine/       ← 検疫ディレクトリ
    │   └── .security-audit-progress.md ← 監査進捗（自動生成）
    ├── plugins/
    └── uploads/
```

---

## スキル一覧

| コマンド | 用途 | いつ使う |
|---------|------|---------|
| `/security-audit` | **総合監査**（全フェーズ・進捗追跡付き） | まず最初にこれを実行 |
| `/malware-scan [パス]` | 危険パターンのスキャン | 素早くスキャンしたい時 |
| `/malware-clean [ファイル]` | 検出済みマルウェアを検疫・削除 | スキャン後に実行 |
| `/db-scan [.sqlファイル]` | **DB 検査**（不審アカウント・スクリプト注入・設定改ざん） | 移行前の DB 検査 |
| `/uploads-scan [ディレクトリ]` | **uploads 検査**（PHP混入・偽装・SVGスクリプト注入） | 移行前の uploads 検査 |
| `/htaccess-security [パス]` | .htaccess のセキュリティ設定をレビュー・追記 | 設定強化したい時 |

## エージェント一覧

| エージェント | 用途 | 使い方 |
|------------|------|--------|
| `malware-investigator` | 疑いファイルの深掘り分析・難読化デコード | 「malware-investigator エージェントで [ファイル名] を分析して」 |
| `site-hardener` | クリーンアップ後のサイト全体堅牢化 | 「site-hardener エージェントを実行して」 |
