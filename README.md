# claude-wp-security

WordPress サイトのマルウェア検出・クリーンアップ・移行を Claude Code で行うためのセキュリティツールキット。

## 概要

感染した WordPress サイトを安全にクリーンアップし、新サーバーへ移行するための Claude Code スキル・エージェント・設定ファイル一式です。

「検査してください」と入力するだけで Claude が自動的に全フェーズの監査を開始します。

## 必要環境

- [Claude Code](https://claude.ai/code)
- WordPress サイト（テーマディレクトリへの配置を推奨）

## インストール

```bash
cd /path/to/wp-content/themes/[your-theme]

git clone https://github.com/kentaChinen/claude-wp-security.git tmp-security
cp tmp-security/CLAUDE.md .
cp -r tmp-security/.claude .
rm -rf tmp-security

CLAUDE.md を開いてプロジェクト概要（サイト名・状態）を記入してください。

使い方

Claude Code を起動して以下のキーワードを入力すると /security-audit が自動起動します：

検査 / 検索 / 調査 / 調べて / 確認して / チェック / スキャン /
ウイルス / マルウェア / 脆弱性 / 感染 / 不正 / セキュリティ

スキルを直接実行することも可能です：

/security-audit            # 総合監査（まず最初にこれを実行）
/malware-scan [パス]       # 危険パターンのスキャン
/malware-clean [ファイル]  # 検出済みマルウェアを検疫・削除
/db-scan [.sqlファイル]    # DB 内のマルウェア・不審アカウント検査
/uploads-scan [ディレクトリ] # uploads の詳細検査
/htaccess-security [パス]  # .htaccess のセキュリティ強化

スキル一覧

┌────────────────────┬────────────────────────────────────────────────────────────────────────────────────┐
│      コマンド      │                                        用途                                        │
├────────────────────┼────────────────────────────────────────────────────────────────────────────────────┤
│ /security-audit    │ 全 9 フェーズの総合監査。進捗を .security-audit-progress.md に保存し途中再開が可能 │
├────────────────────┼────────────────────────────────────────────────────────────────────────────────────┤
│ /malware-scan      │ eval / base64_decode / system 等の危険パターンを網羅的にスキャン                   │
├────────────────────┼────────────────────────────────────────────────────────────────────────────────────┤
│ /malware-clean     │ マルウェアを .quarantine/ に移動して検疫。直接削除は行わない                       │
├────────────────────┼────────────────────────────────────────────────────────────────────────────────────┤
│ /db-scan           │ mysqldump ファイルの wp_options / wp_users / wp_posts を検査                       │
├────────────────────┼────────────────────────────────────────────────────────────────────────────────────┤
│ /uploads-scan      │ PHP 混入・拡張子偽装・SVG スクリプト注入を検査                                     │
├────────────────────┼────────────────────────────────────────────────────────────────────────────────────┤
│ /htaccess-security │ WP ルート・テーマ・uploads の .htaccess を一括レビュー・追記                       │
└────────────────────┴────────────────────────────────────────────────────────────────────────────────────┘

エージェント一覧

┌──────────────────────┬─────────────────────────────────────────────────────────────────────┐
│     エージェント     │                                用途                                 │
├──────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ malware-investigator │ 疑いファイルの難読化デコード・攻撃手法の解析                        │
├──────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ site-hardener        │ クリーンアップ後のパーミッション・wp-config・functions.php の堅牢化 │
└──────────────────────┴─────────────────────────────────────────────────────────────────────┘

移行ワークフロー

感染サーバーから新サーバーへ安全に移行する手順：

STEP1. 感染サーバーから テーマ / DB / uploads をダウンロード
          ↓
STEP2. ローカルで検査・クリーンアップ
        /security-audit  → テーマファイル検査
        /db-scan         → DB 検査
        /uploads-scan    → uploads 検査
        functions.php にセキュリティコードを追加
          ↓
STEP3. 再検査（全ファイルを改めて確認）
          ↓
STEP4. 新サーバーへアップ
        WP 新規インストール + プラグイン新規インストール
        /htaccess-security で .htaccess 強化
        wp-config.php にシークレットキー再生成・DISALLOW_FILE_EDIT 設定
          ↓
STEP5. DNS 切り替え・WAF 導入・最終確認

ディレクトリ構成

[テーマルート]/
├── CLAUDE.md                          # プロジェクトコンテキスト・行動ルール
└── .claude/
    ├── settings.json                  # Bash コマンド許可・禁止設定
    ├── security-audit/SKILL.md        # /security-audit
    ├── malware-scan/SKILL.md          # /malware-scan
    ├── malware-clean/SKILL.md         # /malware-clean
    ├── db-scan/SKILL.md               # /db-scan
    ├── uploads-scan/SKILL.md          # /uploads-scan
    ├── htaccess-security/SKILL.md     # /htaccess-security
    └── agents/
        ├── malware-investigator.md
        └── site-hardener.md

セキュリティポリシー

- マルウェアファイルは直接削除せず必ず .quarantine/YYYY-MM-DD/ へ移動
- WordPress コアファイル（wp-includes/ wp-admin/ wp-*.php）は変更しない
- 疑いコードは eval / include / require で実行しない（Read ツールで確認のみ）
- ファイル移動・削除・.htaccess 書き換えは必ずユーザー確認後に実行

ライセンス

MIT
```
