# デプロイ情報（DEPLOYMENT）

## Google Apps Script ウェブアプリ

- デプロイURL: `https://script.google.com/macros/s/AKfycbxSDCo0qQHkpoGH5sSk8wJIrfEzPekB-LXCv85FVCgzbirz8mc3nxVd8IuKBjN9h6cn/exec`
- 次のユーザーとして実行: 自分
- アクセスできるユーザー: 全員
- 初回OAuth承認: 完了（Hiroさん本人により実施済み）
- 実装コード: `docs/requirements.md` §6 記載の `doPost` 確定版（Code.gs）

### 再デプロイ時の注意
コード修正時は「デプロイを管理」→「新バージョン」で更新する。URLは変わらないため、フロント側（`marketing_dna_test.html`内の`sendResult()`）の貼り替えは不要。

## Googleスプレッドシート

- ファイル名: マーケティングDNAテスト_結果
- シート名: `シート1`
- ヘッダー（12列）: 日時 / お名前 / A / B / C / D / E / F / top1 / top2 / タイプコード / タイプ名
- 共有設定: 非公開（Hiroさん専用。編集者リンクは配布しない）

## GitHub Pages

- リポジトリ: https://github.com/hiro-artis/marketing-dna-test（Private）
- Pages公開: Phase 12で実施予定（未実施）
