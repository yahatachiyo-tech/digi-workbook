# GAS ログ送信が機能しない問題の調査と修正

**作成日**: 2026-04-26
**症状**: ダッシュボードに「本日ログイン 0/36」と表示され、生徒のログイン状況が反映されない

---

## 調査結果

### Playwright での動作確認

1. **フロントエンド側**: ログイン時の POST リクエストは正常に発火している
   ```
   POST https://script.google.com/macros/s/AKfycbz.../exec
   Content-Type: text/plain;charset=utf-8
   Body: {"timestamp":"2026-04-26T13:40:12.775Z","student_id":1,"unit_id":"","problem_id":"","stage":"login",...}
   Response: 302 → 200 (リダイレクト後)
   ```

2. **GAS doGet の動作**: パラメータ付き GET は正常応答
   ```
   curl 'https://...?key=mdesign2026'
   → {"logs":[]}
   ```

3. **問題**: POST が GAS に届いているのに、Spreadsheet に書き込まれていない（読み出すと空）

---

## 想定される原因

優先度順:

### A) GAS doPost が `e.postData.contents` を読めていない

HTML から `Content-Type: text/plain;charset=utf-8` で送信しているため、GAS 側では `e.parameter` ではなく **`e.postData.contents`** を使う必要があります。

### B) Spreadsheet のヘッダー行と doPost の書き込みカラム順が不一致

doPost で `appendRow([...])` する際、Spreadsheet のヘッダー（1 行目）と異なるカラム順で追加すると、データが間違った列に入る → `timestamp` カラムが空 → doGet の `if (!ts) return false;` で除外される。

### C) doPost で例外発生 → silent fail

GAS 側のログ実行時例外（型エラー等）でデータ書き込みが行われない。

---

## 推奨される doPost 実装

GAS の Apps Script editor で、既存の doPost を以下に置き換えてください（または新規追加）。

```javascript
// === ログ受信用 doPost（2026-04-26 改訂版）===
//   フロントエンド（HTML）から JSON ボディで POST されたログを Spreadsheet に追記。
//   Content-Type: text/plain;charset=utf-8 で送信されるため e.postData.contents を使用。
//   timestamp は ISO 文字列（UTC）として記録される。
//   appendRow ではなく、ヘッダー名でカラムを動的に対応付けることで、
//   ヘッダー順変更にも耐性のある実装にする。

function doPost(e) {
  try {
    // ボディをパース
    if (!e || !e.postData || !e.postData.contents) {
      return ContentService.createTextOutput('error: no body')
        .setMimeType(ContentService.MimeType.TEXT);
    }
    const data = JSON.parse(e.postData.contents);

    const ss = SpreadsheetApp.getActiveSpreadsheet();
    const sheet = ss.getSheets()[0];  // または sheet 名で指定: ss.getSheetByName('シート1')

    // ヘッダー行を取得（1 行目）
    const headers = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];

    // ヘッダーが空なら初期化
    if (!headers[0]) {
      const defaultHeaders = [
        'timestamp', 'student_id', 'unit_id', 'problem_id', 'stage',
        'is_correct', 'hint_level_used', 'hint_count', 'elapsed_seconds',
        'level', 'answer', 'note'
      ];
      sheet.getRange(1, 1, 1, defaultHeaders.length).setValues([defaultHeaders]);
      headers.length = 0;
      defaultHeaders.forEach((h, i) => headers[i] = h);
    }

    // ヘッダー名 → 値の対応で行を構築
    const row = headers.map(h => (data[h] !== undefined ? data[h] : ''));

    // 末尾に追加
    sheet.appendRow(row);

    return ContentService.createTextOutput('ok')
      .setMimeType(ContentService.MimeType.TEXT);
  } catch (err) {
    // 例外もログとして記録（デバッグ用）
    try {
      const ss = SpreadsheetApp.getActiveSpreadsheet();
      const sheet = ss.getSheets()[0];
      sheet.appendRow([
        new Date().toISOString(),
        '',
        '',
        '',
        'error',
        '',
        'none',
        0, 0,
        '基礎',
        '',
        'doPost error: ' + String(err)
      ]);
    } catch (e2) { /* nested error */ }
    return ContentService.createTextOutput('error: ' + String(err))
      .setMimeType(ContentService.MimeType.TEXT);
  }
}
```

### 既存の doGet も整合性チェック

既存の doGet（GAS_DASHBOARD_SETUP.md に記載）は変更不要ですが、`tsIdx` が `-1` のとき（`timestamp` ヘッダーが存在しない場合）にフォールバックする処理があれば堅牢です。

---

## 修正手順

1. Spreadsheet を開く → 拡張機能 → Apps Script
2. 既存の `doPost` を上記コードで置き換え
3. 保存（Cmd+S）
4. **再デプロイ**（必須）:
   - 「デプロイ」→「デプロイを管理」
   - 既存デプロイの **鉛筆マーク**で編集
   - バージョンを「**新しいバージョン**」に変更
   - 「デプロイ」をクリック
5. 動作確認:
   - デジタルワークでログイン → ダッシュボードを開く
   - 「テスト送信」ボタンをクリック
   - 30 秒後「今すぐ更新」→「生ログ表示」
   - `student_id: 99` の login ログが表示されれば成功

---

## デバッグ用の機能（フロントエンド側）

### `[テスト送信]` ボタン
- 教員モードのダッシュボードに新規追加
- `student_id: 99` で疑似ログイン記録を送信（学生記録と区別可能）
- ブラウザのコンソールに送信内容を出力

### `[生ログ表示]` ボタン
- 教員モードのダッシュボードに新規追加
- GAS から取得した最新の logs JSON をそのまま画面に表示
- 「件数」「実データ」が見える → どの段階で問題があるか判断可能

### `keepalive: true` 追加
- `sendLog` の fetch オプションに追加
- ログイン直後の `goToMenu()` ナビゲーションで送信が中断される問題を防止

---

## まだ動かない場合の追加調査

### 1. GAS の実行履歴を確認

Apps Script editor → 左サイドバー「実行数」→ 最近の `doPost` 実行を確認。
失敗していれば、エラーメッセージが表示される。

### 2. Spreadsheet を直接開いて確認

シート 1 行目のヘッダー列名を確認。
```
A: timestamp
B: student_id
C: unit_id
D: problem_id
E: stage
F: is_correct
G: hint_level_used
H: hint_count
I: elapsed_seconds
J: level
K: answer
L: note
```

これと一致していなければ、上記の改訂版 doPost で自動的にヘッダーを補完します。

### 3. 手動で 1 行追加してみる

Spreadsheet に直接 1 行ダミーデータを追加して、doGet が拾うかテスト:
```
2026-04-26T22:00:00.000Z | 1 | | | login | | none | 0 | 0 | 基礎 | | manual-test
```

「今すぐ更新」→「生ログ表示」で `manual-test` のログが見えれば、doGet は正常。
書き込みが doPost 経由で失敗しているだけと特定できる。

---

## セキュリティ補足

- 教員モードのテスト送信は `student_id: 99` を使うため、生徒のデータと混在しない
- 実運用時に `student_id: 99` のログが大量に残っている場合は、Spreadsheet で手動削除推奨
