# マーケティングDNAテスト（Marketing DNA Test Japan v2.0）

> 設計の共通原則（基本原則・資産価値の原則）は `~/.claude/CLAUDE.md` に従う。

## プロジェクト概要

スタッフ向けマーケティングDNA診断テスト（単一HTMLファイル）。LINE個人共有／会場QRコードで配布し、結果をGoogle Apps Script経由でGoogleスプレッドシートに自動収集する。詳細は `docs/requirements.md`。

## プロジェクト設定

技術スタック:
  frontend: 単一HTML/CSS/バニラJS（`marketing_dna_test.html`。フレームワーク・ビルドツールなし）
  hosting: GitHub Pages（固定URL）
  backend: Google Apps Script ウェブアプリ（`doPost`）
  database: Googleスプレッドシート（Hiro専用共有）

このプロジェクトにはビルドプロセス・パッケージマネージャ・開発サーバーは存在しない。ローカル確認は `marketing_dna_test.html` をブラウザで直接開くか、簡易HTTPサーバー（例: `python3 -m http.server`）で配信して行う（`file://` 直開きだと後述のGAS `fetch` がCORSで失敗するため、確認時は必ずHTTPサーバー経由にする）。

## GASウェブアプリURLの扱い

GASデプロイURL（`https://script.google.com/macros/s/XXXX/exec`）は「アクセスできるユーザー＝全員」で公開する設計上の性質から、秘匿情報ではない。`marketing_dna_test.html` 内に直接記述してよい（キーチェーン管理・環境変数化は不要）。

## 開発ルール

### コード品質
- 関数: 100行以下 / ファイル: 700行以下 / 複雑度: 10以下 / 行長: 120文字
- 本プロジェクトはビルドツールを持たないため、ESLint等の自動Lintは導入せず、上記基準は目視レビューで担保する（過剰なツール導入は資産価値の原則「過剰設計はしない」に反するため見送り）

### 変更時の注意
- `marketing_dna_test.html` の診断ロジック（`LIKERT` / `ABILITY` / `TYPES` / `computeScores()` / `getTypeCode()`）は完成済み。今回のスコープ（名前入力画面・GAS自動送信）以外は変更しない
- GAS側コード（`Code.gs`）を修正した場合は「デプロイを管理」→「新バージョン」で更新する。URLは変わらないため、フロント側の貼り替えは不要

### デプロイ
- デプロイ（GitHub Pagesへの公開・GASの新バージョン公開）はユーザーの明示的な承認を得てから実行する
- Google Apps Scriptの初回「アクセスを承認」（OAuth同意）は必ずHiroさん本人が実施する

### テスト
- 実機テストは必須（`docs/requirements.md` §9）: PCブラウザ → LINE内ブラウザ → `?openExternalBrowser=1`付きリンクの3パターンを確認してから本番配布する

### ドキュメント管理
許可されたドキュメントのみ作成可能:
- docs/SCOPE_PROGRESS.md（実装計画・進捗）
- docs/requirements.md（要件定義）
- docs/DEPLOYMENT.md（デプロイ情報）
上記以外のドキュメント作成はユーザー許諾が必要。

## Git/CI設定

### 品質ゲート
本プロジェクトはビルドツールを持たないため、GitHub Actions等の自動CIは設けない。品質は目視レビューで担保する（上記「コード品質」節を参照）。

### ローカルフック（`.git/hooks/`）
- `prepare-commit-msg`: コミットメッセージ冒頭に日時を自動追加
- `pre-commit`: `.env`系・秘密鍵ファイルがstageされていた場合コミットを中止

### ブランチ戦略
- `main`: 本番（GitHub Pages公開ブランチ）。force push・削除を禁止（branch protection）
- `develop`: 開発統合ブランチ

### リポジトリ
- URL: https://github.com/hiro-artis/marketing-dna-test
- 公開設定: Private
