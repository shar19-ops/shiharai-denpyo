# 添付ファイル機能 設計

日付: 2026-07-24
対象: `一般管理費支払伝票アプリ_v3`(支払伝票【一般管理費】.html)、`payment-slip-app`(支払伝票【未成工事用】.html)の両方に同じ内容を実装する。

## 背景・目的

領収書(写真、自動トリミング機能)とは別に、PDFファイル(見積書・仕様書など)を1件以上添付できるようにしたい。印刷/PDF出力時に、伝票本体・領収書ページの後ろに添付PDFのページをそのまま結合して出力する。

参考実装: `transfer-slip-app`(振替伝票.html)の「請求書PDF添付」機能。ただし振替伝票は1件のみ対応のため、今回は複数件に対応させる。

## データ構造

```javascript
let attachedFiles = []; // [{ name, dataUrl }]
```

`receiptImages`(既存の領収書配列)と同じ配列パターン。`dataUrl`はFileReaderの`readAsDataURL`結果(`data:application/pdf;base64,...`)。

## UI

領収書カードの直後に新規カードを追加する。

```html
<!-- 添付ファイル -->
<div class="card">
  <div class="card-header">▌ 添付ファイル</div>
  <div style="padding:14px 16px;">
    <input type="file" id="attached-file-input" accept="application/pdf,.pdf" multiple style="display:none" onchange="onAttachedFilesSelected(this)">
    <button type="button" class="btn-scan" onclick="document.getElementById('attached-file-input').click()">📎 ファイルを添付</button>
    <div id="attached-file-list"></div>
  </div>
</div>
```

`attached-file-list`内に、添付ファイルごとに1行(ファイル名 + 👁 表示ボタン + 🗑 削除ボタン)を描画する。0件のときはリストを空表示(領収書のように「未添付」文言は必須ではない)。

## 主要関数

- `onAttachedFilesSelected(input)`: 選択された各ファイルについて、PDF判定(`file.type === 'application/pdf'` または拡張子`.pdf`)→ 不正なら該当ファイルだけ`alert`でスキップ → `FileReader.readAsDataURL`で読み込み → `attachedFiles`に`{name, dataUrl}`を追加 → 一覧再描画。複数ファイル選択時は各ファイルを個別に非同期処理する。
- `removeAttachedFileAt(idx)`: 該当indexを配列から削除して再描画。
- `viewAttachedFile(idx)`: 振替伝票の`viewInvoicePdf`と同様、dataURLをBlob化してblob: URLを新規タブで開く。
- `renderAttachedFileList()`: `attached-file-list`の中身を再構築。

## 保存・読み込み(下書きJSON)

`collectFormData()`に`__attachedFiles: attachedFiles`を追加。
`applyFormData()`で`attachedFiles = data.__attachedFiles || []`を復元し、`renderAttachedFileList()`を呼ぶ。

## クリア

`clearForm()`内で`attachedFiles = []`にして`renderAttachedFileList()`を呼ぶ。

## PDF出力時の結合

既存の出力処理(伝票本体描画 → 領収書ページ追加)の後に追加する。

```javascript
let attachMergeWarnings = [];
for (const f of attachedFiles) {
  try {
    const bytes = Uint8Array.from(atob(f.dataUrl.split(',')[1]), c => c.charCodeAt(0));
    const doc = await PDFDocument.load(bytes);
    const copiedPages = await a4Doc.copyPages(doc, doc.getPageIndices());
    copiedPages.forEach(page => a4Doc.addPage(page));
  } catch (e) {
    attachMergeWarnings.push(f.name);
  }
}
```

1件の結合失敗が他のファイルや伝票本体の出力を止めないよう、ループ内でtry/catchする(振替伝票は1件だけなので全体catchだったが、複数件対応のため個別化)。失敗があれば`statusEl`のメッセージに「(添付ファイルの結合に失敗: ファイル名, ...)」を付記する(印刷・PDF出力どちらのモードでも同様)。

## 対象外・非スコープ

- ファイルサイズの上限チェックは行わない(既存の領収書機能・振替伝票の添付機能にもない)。
- PDF以外のファイル形式には対応しない。
- ドラッグ&ドロップには対応しない(振替伝票と同じくファイル選択ダイアログのみ)。

## テスト方針

自動テストの仕組みがないアプリのため、Playwrightで実ブラウザ操作(ファイル添付→保存/読み込み→PDF出力→添付PDFのページが結合されていることを確認)を手動で検証する。
