# マーケティングDNAテスト Japan v2.0 — 結果自動収集 要件定義書

## 0. アーキ構成

- 確定アーキ: #6 WEBアプリ（決定論）
- 操作者: ブラウザのエンドユーザー（スタッフ、LINE/QR経由）
- AI本体(プロンプトY): なし（診断ロジックは完全に決定論的な採点＋テーブルルックアップ）
- MCP: なし
- 種別: Web型
- 自社DB: あり（Googleスプレッドシートをデータストアとして使用）
- フロントUI: あり（既存の単一HTMLファイル `marketing_dna_test.html`）
- 動的変数: なし
- 配布: 該当なし（操作者=ブラウザのため、マイ登録/テナント配布の概念は適用外）

---

## 1. プロジェクト概要

### 成果目標
スタッフ向けマーケティングDNA診断テスト（A〜F 6因子診断・36問・15タイプ判定）を、LINE個人共有およびワークショップ会場QRコードで配布し、各回答者の結果を管理者（Hiroさん）専用のGoogleスプレッドシートへ自動収集する。

### 前提（既存実装）
診断ロジック・UI・15タイプ辞書・スコアリング・結果表示・シェア機能は `marketing_dna_test.html` に実装済みであり、本要件定義の対象外（変更しない）。

### 今回の追加スコープ（この2点のみ）
1. テスト開始前の**名前入力画面**の追加（必須入力）
2. `computeAndShowResult()` 実行時に、Google Apps Script（GAS）ウェブアプリへ結果を自動送信する機能の追加

### 成功指標
**定量的指標**
- LINE内ブラウザからの送信成功率100%（実機テストで検証）
- スプレッドシートへの反映が送信後数秒以内
- 初期構築費・月額運用費ともに0円
- 数十〜数百名規模でも遅延・エラーなく処理できる

**定性的指標**
- 回答者は自分の結果しか見えず、安心してテストを受けられる
- Hiroさん以外は誰も集計データにアクセスできない
- LINEでリンク／QRを受け取るだけで、ログイン等の手間なくテストを完了できる
- 診断結果がひと目で分かりやすく表示される（既存UIのまま）

---

## 2. システム全体像

### 業務フロー
```
① スタッフがLINE個人共有 or 会場QRコードでアプリURLを受け取る
② 【新規】名前を入力する（必須。本名・ニックネームどちらも可。「お名前（ニックネームでも可）」ラベルで気軽に入力できるようにする）
③ 診断テストに回答する（30問リッカート＋能力テスト6問／既存）
④ 結果が画面に表示される（A〜Fスコア・15タイプ判定・戦略ステップ／既存）
⑤ 【新規】結果表示と同時に、バックグラウンドでGoogleスプレッドシートへ自動送信（画面遷移なし）
⑥ Hiroさんがスプレッドシートで全員の結果を確認する
```

### アクセス制御（ロール設計なし）
本アプリにログイン機能・ロール分けは存在しない。

| 対象 | アクセス可能な人 |
|---|---|
| 診断アプリ本体（URL/QR） | リンクを知っている全員（スタッフ） |
| 自分の結果表示 | 回答者本人のみ（ローカル表示・送信はしない） |
| Googleスプレッドシート（全回答集計） | Hiroさんのみ（非公開共有・編集者リンクは配布しない） |

---

## 3. 画面詳細仕様

| ID | 画面名 | 目的 | 変更内容 |
|---|---|---|---|
| S-00 | ウェルカム | サービス概要・開始導線 | 既存のまま |
| S-01 | **名前入力**（新規） | 回答者識別のための名前必須入力 | `screen-welcome` の「診断をはじめる」ボタン押下後、`startTest()` が呼ばれる前に新規画面を挿入。空欄では次へ進めない。**本名・ニックネームどちらも可**（所属欄は設けない。誰か特定できれば十分とし、入力の気軽さを最優先。入力欄ラベルは「お名前（ニックネームでも可）」と表記する） |
| S-02 | リッカート調査（30問） | 因子スコア収集 | 既存のまま |
| S-03 | 能力テスト導入 | Part2説明 | 既存のまま |
| S-04 | 能力テスト（6問） | タイブレーク用スコア収集 | 既存のまま |
| S-05 | 結果表示 | スコア・タイプ・戦略表示 | 表示自体は既存のまま。`computeAndShowResult()` 内で `sendResult()` を1回だけ呼び出す |

### 実装方針（フロントエンド側）

1. `state` オブジェクトに `name: ''` を追加する。
2. `#screen-welcome` の次に `#screen-name` を新設し、`startTest()` の呼び出し元を「名前入力完了ボタン」に付け替える。
3. 名前入力欄は `required`。空欄時はボタンを disabled にするか、送信時にアラート表示して遷移させない。
4. `computeAndShowResult()` の冒頭で `sendResult(state.name, scores, code, type)` を呼び出す（結果表示をブロックしない = 非同期・fire-and-forgetで実行し、失敗してもUI表示は継続する）。
5. 二重送信防止のため、送信済みフラグ（例: `state.sent`）を立てて多重呼び出しを防ぐ。
6. **送信結果のUI表示（確定仕様）**：成功時は何も表示しない（サイレント）。失敗時（`fetch`の`catch`、またはレスポンス`result:'error'`）のみ、結果画面（S-05）下部に控えめなインライン注意書きを表示する：「※結果の記録に失敗しました。この画面のスクリーンショットを管理者に送ってください」。`alert()`等のエラーダイアログは使用しない。

---

## 4. データ設計概要

### 送信データ（GASへPOSTするJSON）
`computeScores()` の戻り値（`avgs`, `top1`, `top2`）と `getTypeCode()` / `TYPES` の結果から構成する。

| フィールド | 型 | 由来 |
|---|---|---|
| name | string | 新規追加の名前入力欄 |
| A〜F | number（小数1桁） | `scores.avgs.A`〜`F` |
| top1 / top2 | string（因子キー） | `scores.top1` / `scores.top2` |
| code | string（2文字） | `getTypeCode(scores.top1, scores.top2)`（例: `BE`） |
| typeName | string | `TYPES[code].name`（例: `パフォーマー型`） |

### スプレッドシート列構成
```
日時 / お名前 / A / B / C / D / E / F / top1 / top2 / タイプコード / タイプ名
```
（日時は `new Date()` としてGAS側 `appendRow` 時に付与する）

### データストア
Googleスプレッドシート（Hiroさん個人アカウント所有・非公開共有）。新規のPostgres/Supabase等は不要。

---

## 5. セキュリティ要件

- 通信は全てHTTPS（GitHub Pages / GAS ともに標準でHTTPS強制）
- 名前入力欄は送信前にトリム・簡易サニタイズ（HTMLタグ除去程度で十分。GAS側は`appendRow`する文字列としてのみ扱うため実行リスクはない）
- スプレッドシートはHiroさん専用共有。編集者リンクは一切配布しない
- GAS Webアプリのデプロイは「次のユーザーとして実行＝自分」「アクセスできるユーザー＝全員（匿名含む）」とし、Googleアカウントによるログインを一切要求しない構成にする（LINE内ブラウザの`disallowed_useragent`エラー回避のため必須）
- 二重送信・同時書き込み対策として、GAS側で `LockService` を使用する
- 本プロジェクトはビルドプロセス・サーバープロセスを持たないため、ヘルスチェックエンドポイント／グレースフルシャットダウンは対象外

---

## 6. 技術スタック

| 層 | 内容 |
|---|---|
| フロントエンド | 既存の単一HTML/CSS/バニラJS（フレームワーク追加なし） |
| ホスティング | GitHub Pages（固定URL。git push で更新反映） |
| バックエンド | Google Apps Script ウェブアプリ（`doPost`、方式A：`Content-Type: text/plain` でCORSプリフライト回避） |
| データベース | Googleスプレッドシート（Hiro専用共有） |

### GAS側実装（Code.gs・設計確定版）

```javascript
function doPost(e) {
  var lock = LockService.getScriptLock();
  lock.tryLock(10000);
  try {
    var sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('シート1');
    var data = JSON.parse(e.postData.contents);
    sheet.appendRow([
      new Date(),
      data.name, data.A, data.B, data.C, data.D, data.E, data.F,
      data.top1, data.top2, data.code, data.typeName
    ]);
    return ContentService.createTextOutput(JSON.stringify({ result: 'ok' }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService.createTextOutput(JSON.stringify({ result: 'error', message: String(err) }))
      .setMimeType(ContentService.MimeType.JSON);
  } finally {
    lock.releaseLock();
  }
}
```

デプロイ設定：「次のユーザーとして実行＝自分」／「アクセスできるユーザー＝全員」。

### フロントエンド側送信コード（設計確定版）

```javascript
function sendResult(name, scores, code, type) {
  if (state.sent) return;
  state.sent = true;
  fetch('GASデプロイURL/exec', {
    method: 'POST',
    headers: { 'Content-Type': 'text/plain;charset=utf-8' },
    body: JSON.stringify({
      name: name,
      A: scores.avgs.A, B: scores.avgs.B, C: scores.avgs.C,
      D: scores.avgs.D, E: scores.avgs.E, F: scores.avgs.F,
      top1: scores.top1, top2: scores.top2,
      code: code, typeName: type.name,
    }),
  })
    .then(function (res) { return res.json(); })
    .then(function (json) {
      if (json.result !== 'ok') showSendError();
    })
    .catch(function () { showSendError(); });
}

function showSendError() {
  // 成功時は何も表示しない。失敗時のみ結果画面下部に控えめな注意書きを出す（alert等は使わない）
  var el = document.getElementById('send-error-note');
  if (el) el.style.display = 'block';
}
```

結果画面（S-05）のテンプレートに、初期状態で非表示の注意書き要素を追加する：
```html
<p id="send-error-note" style="display:none; font-size:13px; color:var(--ink-faint); margin-top:16px;">
  ※結果の記録に失敗しました。この画面のスクリーンショットを管理者に送ってください。
</p>
```

GASデプロイURLは環境変数ではなく、公開ウェブアプリのエンドポイントURL（秘匿情報ではない）としてコード中に直接記述してよい。

---

## 7. 外部サービス一覧

| API/サービス | 用途 | 選定理由 |
|---|---|---|
| Google Apps Script | 結果データ自動収集エンドポイント | サーバー不要・無料・Googleネイティブ・LINE内ブラウザでもログイン不要 |
| GitHub Pages | 静的ホスティング | 無料・恒久URL・git pushで更新可能・QRコード化に適した固定URL |

---

## 8. 配布・運用

- 配布リンクには `?openExternalBrowser=1` を付与し、LINE内ブラウザではなく外部ブラウザ（Safari/Chrome）で開かせる
- ワークショップ会場向けにQRコードを作成（GitHub Pages固定URLをQR化）
- 新メンバー参加の都度、継続的に実施する運用（一度きりではない）
- コードを修正した場合、GAS側は「新バージョン」として再デプロイ（URLは変わらないためフロント側の貼り替え不要）

---

## 9. 実機テスト計画（必須）

1. PCブラウザで送信 → スプレッドシートに1行増えるか確認
2. スマホのLINEで自分にリンクを送り、LINE内ブラウザで開いて送信 → 書き込まれるか確認（必須。GAS「全員」デプロイ×LINE WebViewの組み合わせの実地検証事例が少ないため、本番配布前に必ずHiroさん自身が実施）
3. `?openExternalBrowser=1` 付きリンクで外部ブラウザに退避されるか確認
