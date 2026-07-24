# 添付ファイル機能 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 領収書(写真)とは別に、PDFファイルを複数件添付できる機能を`payment-slip-app`(未成工事用)と`一般管理費支払伝票アプリ_v3`(一般管理費)の両方に追加し、印刷/PDF出力時に伝票・領収書ページの後ろへ添付PDFのページを結合する。

**Architecture:** 両アプリともビルド無しの単一HTMLファイル(フレームワーク不使用、pdf-lib`PDFLib`をグローバルで使用)。`receiptImages`配列と同じパターンで`attachedFiles`配列を追加し、専用のUIカード・関数群・PDF生成時のマージ処理を実装する。参考実装は`transfer-slip-app`(振替伝票.html)の請求書PDF添付機能(1件のみ)だが、今回は複数件対応にする。

**Tech Stack:** 素のJavaScript(ES2017+)、pdf-lib(`PDFLib`グローバル、CDN読み込み済み)、Playwright(検証用)。

## Global Constraints

- 対応ファイル形式はPDFのみ(`file.type === 'application/pdf'` または拡張子`.pdf`で判定)。
- ファイルサイズの上限チェックは行わない。
- ドラッグ&ドロップ非対応、ファイル選択ダイアログのみ。
- このリポジトリには自動テストの仕組み(pytest/jest等)が無い。各タスクの「テスト」はPlaywright(`mcp__plugin_playwright_playwright__*`ツール)による実ブラウザでの動作検証に置き換える。検証は不合格なら次のタスクに進まない。
- 両アプリの対象ファイルは以下の通り(パスはWindows形式、Bashツールからは`/c/Users/...`形式でアクセス可能):
  - `C:\Users\shar1\OneDrive\MCフォルダ\支払伝票PJ\payment-slip-app\支払伝票【未成工事用】.html`
  - `C:\Users\shar1\OneDrive\MCフォルダ\支払伝票PJ\一般管理費支払伝票アプリ_v3\支払伝票【一般管理費】.html`
- 実装後、変更したファイルのあるリポジトリでgit commitする(pushはユーザーの明示的な指示があってから)。

---

## 検証用ローカルサーバーの起動方法(全タスク共通)

各タスクの検証では、対象アプリのフォルダを簡易HTTPサーバーで配信してPlaywrightからアクセスする。Node.js標準の`http`モジュールのみで動く以下のスクリプトを使う。

`C:\Users\shar1\AppData\Local\Temp\claude\static-server.js`(無ければ作成する):

```javascript
const http = require('http');
const fs = require('fs');
const path = require('path');
const url = require('url');

const root = process.argv[2];
const port = Number(process.argv[3] || 8900);

const mime = {
  '.html': 'text/html; charset=utf-8',
  '.js': 'text/javascript; charset=utf-8',
  '.pdf': 'application/pdf',
};

http.createServer((req, res) => {
  const parsed = url.parse(req.url);
  let p = decodeURIComponent(parsed.pathname);
  if (p === '/') p = '/index.html';
  const filePath = path.join(root, p);
  fs.readFile(filePath, (err, data) => {
    if (err) { res.writeHead(404); res.end('not found'); return; }
    const ext = path.extname(filePath).toLowerCase();
    res.writeHead(200, { 'Content-Type': mime[ext] || 'application/octet-stream' });
    res.end(data);
  });
}).listen(port, () => console.log('listening on ' + port));
```

起動コマンド(payment-slip-appの例、ポート8900):

```bash
node "C:\Users\shar1\AppData\Local\Temp\claude\static-server.js" "/c/Users/shar1/OneDrive/MCフォルダ/支払伝票PJ/payment-slip-app" 8900 &
```

アプリURL(payment-slip-app、未成工事用):
`http://localhost:8900/%E6%94%AF%E6%89%95%E4%BC%9D%E7%A5%A8%E3%80%90%E6%9C%AA%E6%88%90%E5%B7%A5%E4%BA%8B%E7%94%A8%E3%80%91.html`

一般管理費支払伝票アプリ_v3を検証する際は、フォルダとポートを変えて別途起動する(例: ポート8902、URLは`支払伝票【一般管理費】.html`をURLエンコードしたもの)。

`generatePDF`はネット上のフォント(`https://raw.githubusercontent.com/google/fonts/main/ofl/mplus1p/MPLUS1p-Regular.ttf`)を取得するため、検証環境でこのURLに到達できない場合は失敗して英数字のみのフォールバックフォントになるが、添付PDF結合機能の検証自体には影響しない(結合対象はPDFバイト列のコピーであり、フォントを使わない)。

---

## Task 1: payment-slip-app — 添付ファイルUI・状態・操作関数

**Files:**
- Modify: `C:\Users\shar1\OneDrive\MCフォルダ\支払伝票PJ\payment-slip-app\支払伝票【未成工事用】.html:546-547`(HTMLカード追加)
- Modify: `C:\Users\shar1\OneDrive\MCフォルダ\支払伝票PJ\payment-slip-app\支払伝票【未成工事用】.html:1370-1371`(JS関数追加)

**Interfaces:**
- Produces: `attachedFiles`(グローバル配列 `[{name, dataUrl}]`)、`onAttachedFilesSelected(input)`、`removeAttachedFileAt(idx)`、`viewAttachedFile(idx)`、`renderAttachedFileList()`。Task 2がこれらを使用する。

- [ ] **Step 1: 領収書カードの直後に添付ファイルカードのHTMLを追加する**

現在の546-548行目:
```html
    <input type="file" id="receipt-camera-input" accept="image/*" capture="environment" style="display:none;" onchange="handleReceiptInput(this)">
  </div>

  <!-- 発行情報 -->
```

これを以下に置き換える(546行目の直後に新規カードを挿入):
```html
    <input type="file" id="receipt-camera-input" accept="image/*" capture="environment" style="display:none;" onchange="handleReceiptInput(this)">
  </div>

  <!-- 添付ファイル -->
  <div class="card">
    <div class="card-header">▌ 添付ファイル</div>
    <div style="padding:14px 16px;">
      <input type="file" id="attached-file-input" accept="application/pdf,.pdf" multiple style="display:none" onchange="onAttachedFilesSelected(this)">
      <button type="button" class="btn-scan" onclick="document.getElementById('attached-file-input').click()">📎 ファイルを添付</button>
      <div id="attached-file-list"></div>
    </div>
  </div>

  <!-- 発行情報 -->
```

- [ ] **Step 2: 添付ファイル操作用のJS関数を追加する**

現在の1369-1371行目:
```javascript
}

// ─── 領収書スキャン（β版） ────────────────────────────────
```

これを以下に置き換える(1370行目の空行の位置に新セクションを挿入):
```javascript
}

// ─── 添付ファイル ────────────────────────────────────────────
let attachedFiles = []; // [{ name, dataUrl }]

function onAttachedFilesSelected(input) {
  const files = Array.from(input.files || []);
  input.value = '';
  files.forEach(file => {
    const isPdf = file.type === 'application/pdf' || file.name.toLowerCase().endsWith('.pdf');
    if (!isPdf) {
      alert(`「${file.name}」はPDFファイルではないためスキップしました。`);
      return;
    }
    const reader = new FileReader();
    reader.onload = () => {
      attachedFiles.push({ name: file.name, dataUrl: reader.result });
      renderAttachedFileList();
    };
    reader.onerror = () => alert(`「${file.name}」の読み込みに失敗しました。`);
    reader.readAsDataURL(file);
  });
}

function removeAttachedFileAt(idx) {
  attachedFiles.splice(idx, 1);
  renderAttachedFileList();
}

function viewAttachedFile(idx) {
  const f = attachedFiles[idx];
  if (!f) return;
  const bytes = Uint8Array.from(atob(f.dataUrl.split(',')[1]), c => c.charCodeAt(0));
  const url = URL.createObjectURL(new Blob([bytes], { type: 'application/pdf' }));
  const win = window.open(url, '_blank');
  if (!win) alert('ポップアップがブロックされました。ブラウザの設定を確認してください。');
}

function renderAttachedFileList() {
  const container = document.getElementById('attached-file-list');
  container.innerHTML = attachedFiles.map((f, idx) => `
    <div style="display:flex; align-items:center; gap:8px; padding:6px 2px; border-bottom:1px solid #eee; font-size:13px;">
      <span style="flex:1; overflow:hidden; text-overflow:ellipsis; white-space:nowrap;">📄 ${f.name}</span>
      <button type="button" class="btn-scan" onclick="viewAttachedFile(${idx})">👁 表示</button>
      <button type="button" class="btn-scan btn-scan-clear" onclick="removeAttachedFileAt(${idx})">🗑 削除</button>
    </div>
  `).join('');
}

// ─── 領収書スキャン（β版） ────────────────────────────────
```

- [ ] **Step 3: Playwrightで動作確認する**

前提: 検証用サーバーをpayment-slip-appフォルダに対してポート8900で起動済み(共通手順を参照)。

1. `mcp__plugin_playwright_playwright__browser_navigate`で `http://localhost:8900/%E6%94%AF%E6%89%95%E4%BC%9D%E7%A5%A8%E3%80%90%E6%9C%AA%E6%88%90%E5%B7%A5%E4%BA%8B%E7%94%A8%E3%80%91.html` を開く。
2. `mcp__plugin_playwright_playwright__browser_run_code_unsafe`で以下を実行し、ダミーPDF(2ページ)をファイル入力に流し込んでchangeイベントを発火させる:

```javascript
async (page) => {
  const result = await page.evaluate(async () => {
    const doc = await PDFLib.PDFDocument.create();
    doc.addPage([200, 200]);
    doc.addPage([200, 200]);
    const bytes = await doc.save();
    const file = new File([bytes], 'sample-attachment.pdf', { type: 'application/pdf' });
    const dt = new DataTransfer();
    dt.items.add(file);
    const input = document.getElementById('attached-file-input');
    input.files = dt.files;
    input.dispatchEvent(new Event('change', { bubbles: true }));
    return 'dispatched';
  });
  await page.waitForTimeout(500); // FileReaderの非同期読み込み待ち
  const listText = await page.$eval('#attached-file-list', el => el.textContent);
  const count = await page.evaluate(() => attachedFiles.length);
  return { result, listText, count };
}
```

期待結果: `count === 1`、`listText`に`sample-attachment.pdf`が含まれる。

3. 削除ボタンの動作確認:

```javascript
async (page) => {
  await page.evaluate(() => removeAttachedFileAt(0));
  const count = await page.evaluate(() => attachedFiles.length);
  const listText = await page.$eval('#attached-file-list', el => el.textContent);
  return { count, listText };
}
```

期待結果: `count === 0`、`listText === ''`。

4. コンソールエラーが無いことを`mcp__plugin_playwright_playwright__browser_console_messages`(level: "warning")で確認する。期待結果: エラー0件。

- [ ] **Step 4: Commit**

```bash
cd "/c/Users/shar1/OneDrive/MCフォルダ/支払伝票PJ/payment-slip-app"
git add "支払伝票【未成工事用】.html"
git commit -m "feat: 添付ファイル(PDF)のUI・追加/削除/表示機能を実装"
```

---

## Task 2: payment-slip-app — 保存/読み込み/クリア連携とPDF出力への結合

**Files:**
- Modify: `C:\Users\shar1\OneDrive\MCフォルダ\支払伝票PJ\payment-slip-app\支払伝票【未成工事用】.html:920-951`(clearForm/collectFormData/applyFormData)
- Modify: `C:\Users\shar1\OneDrive\MCフォルダ\支払伝票PJ\payment-slip-app\支払伝票【未成工事用】.html:1344-1363`(generatePDF内のマージ処理)

**Interfaces:**
- Consumes: Task 1の`attachedFiles`配列、`renderAttachedFileList()`。
- Produces: なし(最終タスク)。

- [ ] **Step 1: clearFormに添付ファイルのリセットを追加する**

現在の920-926行目:
```javascript
function clearForm() {
  if (!confirm('入力内容をすべてクリアしますか？')) return;
  document.querySelectorAll('input, select').forEach(el => { el.value = ''; });
  clearReceipt();
  setDefaultIssueDate();
  updateTotal();
}
```

これを以下に置き換える:
```javascript
function clearForm() {
  if (!confirm('入力内容をすべてクリアしますか？')) return;
  document.querySelectorAll('input, select').forEach(el => { el.value = ''; });
  clearReceipt();
  attachedFiles = [];
  renderAttachedFileList();
  setDefaultIssueDate();
  updateTotal();
}
```

- [ ] **Step 2: collectFormData/applyFormDataに添付ファイルを含める**

現在の929-951行目:
```javascript
function collectFormData() {
  const data = { __receiptImages: receiptImages.map(r => r.dataUrl) };
  document.querySelectorAll('.container input, .container select').forEach(el => {
    if (!el.id) return;
    data[el.id] = (el.type === 'radio' || el.type === 'checkbox') ? el.checked : el.value;
  });
  return data;
}

function applyFormData(data) {
  Object.entries(data).forEach(([id, val]) => {
    const el = document.getElementById(id);
    if (!el) return;
    if (el.type === 'radio' || el.type === 'checkbox') el.checked = val;
    else el.value = val;
  });
  receiptImages = (data.__receiptImages || []).map(dataUrl => ({ dataUrl, originalImg: null }));
  receiptOriginalImg = null;
  renderReceiptThumbs();
  document.getElementById('receipt-manual-btn').style.display = 'none';
  document.getElementById('receipt-clear-btn').style.display = receiptImages.length ? 'inline-flex' : 'none';
  updateTotal();
}
```

これを以下に置き換える:
```javascript
function collectFormData() {
  const data = { __receiptImages: receiptImages.map(r => r.dataUrl), __attachedFiles: attachedFiles };
  document.querySelectorAll('.container input, .container select').forEach(el => {
    if (!el.id) return;
    data[el.id] = (el.type === 'radio' || el.type === 'checkbox') ? el.checked : el.value;
  });
  return data;
}

function applyFormData(data) {
  Object.entries(data).forEach(([id, val]) => {
    const el = document.getElementById(id);
    if (!el) return;
    if (el.type === 'radio' || el.type === 'checkbox') el.checked = val;
    else el.value = val;
  });
  receiptImages = (data.__receiptImages || []).map(dataUrl => ({ dataUrl, originalImg: null }));
  receiptOriginalImg = null;
  renderReceiptThumbs();
  document.getElementById('receipt-manual-btn').style.display = 'none';
  document.getElementById('receipt-clear-btn').style.display = receiptImages.length ? 'inline-flex' : 'none';
  attachedFiles = data.__attachedFiles || [];
  renderAttachedFileList();
  updateTotal();
}
```

- [ ] **Step 3: PDF生成処理に添付ファイルの結合を追加する**

現在の1344-1363行目:
```javascript
    }

    // ─ 保存 ─
    const outBytes = await a4Doc.save();
    if (mode === 'print') {
      await printViaCanvas(outBytes);
      statusEl.textContent = '印刷ダイアログを開きました';
      statusEl.className = 'status-ok';
    } else {
      const blob = new Blob([outBytes], { type: 'application/pdf' });
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      const dateStr = [v('date_year'), v('date_month'), v('date_day')].filter(Boolean).join('-') || 'output';
      a.download = `支払伝票_${dateStr}.pdf`;
      a.click();
      URL.revokeObjectURL(url);
      statusEl.textContent = 'PDF出力完了！';
      statusEl.className = 'status-ok';
    }
```

これを以下に置き換える:
```javascript
    }

    // ─ 添付ファイル: PDFページを結合 ─
    const attachMergeWarnings = [];
    for (const f of attachedFiles) {
      try {
        const attachBytes = Uint8Array.from(atob(f.dataUrl.split(',')[1]), c => c.charCodeAt(0));
        const attachDoc = await PDFDocument.load(attachBytes);
        const copiedPages = await a4Doc.copyPages(attachDoc, attachDoc.getPageIndices());
        copiedPages.forEach(page => a4Doc.addPage(page));
      } catch (e) {
        attachMergeWarnings.push(f.name);
      }
    }
    const attachWarningMsg = attachMergeWarnings.length
      ? `（添付ファイルの結合に失敗: ${attachMergeWarnings.join(', ')}）`
      : '';

    // ─ 保存 ─
    const outBytes = await a4Doc.save();
    if (mode === 'print') {
      await printViaCanvas(outBytes);
      statusEl.textContent = '印刷ダイアログを開きました' + attachWarningMsg;
      statusEl.className = attachWarningMsg ? 'status-err' : 'status-ok';
    } else {
      const blob = new Blob([outBytes], { type: 'application/pdf' });
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      const dateStr = [v('date_year'), v('date_month'), v('date_day')].filter(Boolean).join('-') || 'output';
      a.download = `支払伝票_${dateStr}.pdf`;
      a.click();
      URL.revokeObjectURL(url);
      statusEl.textContent = 'PDF出力完了！' + attachWarningMsg;
      statusEl.className = attachWarningMsg ? 'status-err' : 'status-ok';
    }
```

- [ ] **Step 4: Playwrightで動作確認する(添付→PDF出力→ページ数確認)**

前提: Task 1と同じ検証用サーバーが起動済み。ページを再読み込みしてから実行する(`browser_navigate`で同じURLに再アクセス)。

```javascript
async (page) => {
  // 1. ダミーPDF(3ページ)を添付
  await page.evaluate(async () => {
    const doc = await PDFLib.PDFDocument.create();
    doc.addPage([200, 200]); doc.addPage([200, 200]); doc.addPage([200, 200]);
    const bytes = await doc.save();
    const file = new File([bytes], 'sample-3page.pdf', { type: 'application/pdf' });
    const dt = new DataTransfer();
    dt.items.add(file);
    const input = document.getElementById('attached-file-input');
    input.files = dt.files;
    input.dispatchEvent(new Event('change', { bubbles: true }));
  });
  await page.waitForTimeout(500);

  // 2. 最低限の入力を埋める
  await page.evaluate(() => {
    document.getElementById('office_code').value = '020';
    document.getElementById('issue_year').value = '2026';
    document.getElementById('issue_month').value = '7';
    document.getElementById('issue_day').value = '24';
  });

  // 3. PDF生成(ダウンロード)して保存
  const downloadPromise = page.waitForEvent('download', { timeout: 30000 });
  await page.evaluate(() => generatePDF(false, 'download'));
  const download = await downloadPromise;
  const savePath = 'C:/Users/shar1/AppData/Local/Temp/claude/attach-test-output.pdf';
  await download.saveAs(savePath);
  const statusText = await page.$eval('#status', el => el.textContent);
  return { statusText, savePath };
}
```

期待結果: `statusText`が`'PDF出力完了！'`(警告文言が付いていない)。

4. 生成されたPDFのページ数を確認する(元の伝票が1ページ+添付3ページ=4ページになっているはず):

```javascript
async (page) => {
  const info = await page.evaluate(async (path) => {
    const resp = await fetch('file://' + path).catch(() => null);
    return resp ? 'fetched' : 'cannot-fetch-file-url';
  }, 'C:/Users/shar1/AppData/Local/Temp/claude/attach-test-output.pdf');
  return info;
}
```

`file://`をfetchできない場合は、代わりに`http://localhost:8901/`(別ポートの静的サーバーで同フォルダを配信)経由でPDFを開き、`mcp__plugin_playwright_playwright__browser_navigate`でPDFビューアのページ数表示(`1 / 4`のような表記)を確認する。期待結果: 総ページ数が4。

5. 保存/読み込みの往復確認:

```javascript
async (page) => {
  const before = await page.evaluate(() => JSON.stringify(collectFormData().__attachedFiles.map(f => f.name)));
  await page.evaluate(() => { window.__savedData = collectFormData(); });
  await page.evaluate(() => { attachedFiles = []; renderAttachedFileList(); });
  const emptied = await page.evaluate(() => attachedFiles.length);
  await page.evaluate(() => applyFormData(window.__savedData));
  const after = await page.evaluate(() => JSON.stringify(attachedFiles.map(f => f.name)));
  return { before, emptied, after };
}
```

期待結果: `emptied === 0`、`before === after`(ファイル名`sample-3page.pdf`が復元されている)。

6. `clearForm`実行後に`attachedFiles`が空になることを確認する(確認ダイアログは`page.on('dialog', d => d.accept())`で自動承認する)。

- [ ] **Step 5: Commit**

```bash
cd "/c/Users/shar1/OneDrive/MCフォルダ/支払伝票PJ/payment-slip-app"
git add "支払伝票【未成工事用】.html"
git commit -m "feat: 添付ファイルを保存/読み込み・クリアに対応し、PDF出力時に結合する"
```

---

## Task 3: 一般管理費支払伝票アプリ_v3 — 添付ファイルUI・状態・操作関数

Task 1と同じ内容を、`C:\Users\shar1\OneDrive\MCフォルダ\支払伝票PJ\一般管理費支払伝票アプリ_v3\支払伝票【一般管理費】.html`に適用する。挿入コードはTask 1と同一(ファイル名・関数名も同じ)。挿入位置のみ異なる。

**Files:**
- Modify: `C:\Users\shar1\OneDrive\MCフォルダ\支払伝票PJ\一般管理費支払伝票アプリ_v3\支払伝票【一般管理費】.html:549-550`(HTMLカード追加)
- Modify: `C:\Users\shar1\OneDrive\MCフォルダ\支払伝票PJ\一般管理費支払伝票アプリ_v3\支払伝票【一般管理費】.html:1600-1601`(JS関数追加)

**Interfaces:**
- Produces: Task 1と同じ(`attachedFiles`、`onAttachedFilesSelected`、`removeAttachedFileAt`、`viewAttachedFile`、`renderAttachedFileList`)。Task 4がこれらを使用する。

- [ ] **Step 1: 領収書カードの直後に添付ファイルカードのHTMLを追加する**

現在の548-551行目:
```html
    <input type="file" id="receipt-camera-input" accept="image/*" capture="environment" style="display:none;" onchange="handleReceiptInput(this)">
  </div>

  <!-- 発行情報 -->
```

これを以下に置き換える:
```html
    <input type="file" id="receipt-camera-input" accept="image/*" capture="environment" style="display:none;" onchange="handleReceiptInput(this)">
  </div>

  <!-- 添付ファイル -->
  <div class="card">
    <div class="card-header">▌ 添付ファイル</div>
    <div style="padding:14px 16px;">
      <input type="file" id="attached-file-input" accept="application/pdf,.pdf" multiple style="display:none" onchange="onAttachedFilesSelected(this)">
      <button type="button" class="btn-scan" onclick="document.getElementById('attached-file-input').click()">📎 ファイルを添付</button>
      <div id="attached-file-list"></div>
    </div>
  </div>

  <!-- 発行情報 -->
```

- [ ] **Step 2: 添付ファイル操作用のJS関数を追加する**

現在の1599-1601行目:
```javascript
}

// ─── 領収書スキャン（v3新機能） ────────────────────────────────
```

これを以下に置き換える:
```javascript
}

// ─── 添付ファイル ────────────────────────────────────────────
let attachedFiles = []; // [{ name, dataUrl }]

function onAttachedFilesSelected(input) {
  const files = Array.from(input.files || []);
  input.value = '';
  files.forEach(file => {
    const isPdf = file.type === 'application/pdf' || file.name.toLowerCase().endsWith('.pdf');
    if (!isPdf) {
      alert(`「${file.name}」はPDFファイルではないためスキップしました。`);
      return;
    }
    const reader = new FileReader();
    reader.onload = () => {
      attachedFiles.push({ name: file.name, dataUrl: reader.result });
      renderAttachedFileList();
    };
    reader.onerror = () => alert(`「${file.name}」の読み込みに失敗しました。`);
    reader.readAsDataURL(file);
  });
}

function removeAttachedFileAt(idx) {
  attachedFiles.splice(idx, 1);
  renderAttachedFileList();
}

function viewAttachedFile(idx) {
  const f = attachedFiles[idx];
  if (!f) return;
  const bytes = Uint8Array.from(atob(f.dataUrl.split(',')[1]), c => c.charCodeAt(0));
  const url = URL.createObjectURL(new Blob([bytes], { type: 'application/pdf' }));
  const win = window.open(url, '_blank');
  if (!win) alert('ポップアップがブロックされました。ブラウザの設定を確認してください。');
}

function renderAttachedFileList() {
  const container = document.getElementById('attached-file-list');
  container.innerHTML = attachedFiles.map((f, idx) => `
    <div style="display:flex; align-items:center; gap:8px; padding:6px 2px; border-bottom:1px solid #eee; font-size:13px;">
      <span style="flex:1; overflow:hidden; text-overflow:ellipsis; white-space:nowrap;">📄 ${f.name}</span>
      <button type="button" class="btn-scan" onclick="viewAttachedFile(${idx})">👁 表示</button>
      <button type="button" class="btn-scan btn-scan-clear" onclick="removeAttachedFileAt(${idx})">🗑 削除</button>
    </div>
  `).join('');
}

// ─── 領収書スキャン（v3新機能） ────────────────────────────────
```

- [ ] **Step 3: Playwrightで動作確認する**

Task 1のStep 3と同じ手順を、一般管理費支払伝票アプリ_v3用に起動した検証用サーバー(例: ポート8902、URLは`支払伝票【一般管理費】.html`)に対して実行する。期待結果も同じ(添付後`count===1`かつファイル名が表示、削除後`count===0`、コンソールエラー0件)。

- [ ] **Step 4: Commit**

```bash
cd "/c/Users/shar1/OneDrive/MCフォルダ/支払伝票PJ/一般管理費支払伝票アプリ_v3"
git add "支払伝票【一般管理費】.html"
git commit -m "feat: 添付ファイル(PDF)のUI・追加/削除/表示機能を実装"
```

---

## Task 4: 一般管理費支払伝票アプリ_v3 — 保存/読み込み/クリア連携とPDF出力への結合

**Files:**
- Modify: `C:\Users\shar1\OneDrive\MCフォルダ\支払伝票PJ\一般管理費支払伝票アプリ_v3\支払伝票【一般管理費】.html:1152-1182`(clearForm/collectFormData/applyFormData)
- Modify: `C:\Users\shar1\OneDrive\MCフォルダ\支払伝票PJ\一般管理費支払伝票アプリ_v3\支払伝票【一般管理費】.html:1575-1593`(generatePDF内のマージ処理)

**Interfaces:**
- Consumes: Task 3の`attachedFiles`配列、`renderAttachedFileList()`。
- Produces: なし(最終タスク)。

- [ ] **Step 1: clearFormに添付ファイルのリセットを追加する**

現在の1152-1159行目:
```javascript
function clearForm() {
  if (!confirm('入力内容をすべてクリアしますか？')) return;
  document.querySelectorAll('input, select').forEach(el => { el.value = ''; });
  clearReceipt();
  setDefaultIssueDate();
  updateTotal();
  refreshAllAccountDisplays();
}
```

これを以下に置き換える:
```javascript
function clearForm() {
  if (!confirm('入力内容をすべてクリアしますか？')) return;
  document.querySelectorAll('input, select').forEach(el => { el.value = ''; });
  clearReceipt();
  attachedFiles = [];
  renderAttachedFileList();
  setDefaultIssueDate();
  updateTotal();
  refreshAllAccountDisplays();
}
```

- [ ] **Step 2: collectFormData/applyFormDataに添付ファイルを含める**

現在の1162-1183行目:
```javascript
function collectFormData() {
  const data = { __receiptImages: receiptImages.map(r => r.dataUrl) };
  document.querySelectorAll('.container input, .container select').forEach(el => {
    if (!el.id) return;
    data[el.id] = (el.type === 'radio' || el.type === 'checkbox') ? el.checked : el.value;
  });
  return data;
}

function applyFormData(data) {
  Object.entries(data).forEach(([id, val]) => {
    const el = document.getElementById(id);
    if (!el) return;
    if (el.type === 'radio' || el.type === 'checkbox') el.checked = val;
    else el.value = val;
  });
  receiptImages = (data.__receiptImages || []).map(dataUrl => ({ dataUrl, originalImg: null }));
  receiptOriginalImg = null;
  renderReceiptThumbs();
  document.getElementById('receipt-manual-btn').style.display = 'none';
  document.getElementById('receipt-clear-btn').style.display = receiptImages.length ? 'inline-flex' : 'none';
```
(この後にupdateTotal()やrefreshAllAccountDisplays()等、既存の残りの処理が続く。末尾の`}`の直前に添付ファイル復元処理を挿入する。)

具体的には、`applyFormData`内の以下の行:
```javascript
  document.getElementById('receipt-clear-btn').style.display = receiptImages.length ? 'inline-flex' : 'none';
```
の直後に、次の2行を追加する:
```javascript
  attachedFiles = data.__attachedFiles || [];
  renderAttachedFileList();
```

また`collectFormData`の1行目:
```javascript
  const data = { __receiptImages: receiptImages.map(r => r.dataUrl) };
```
を以下に置き換える:
```javascript
  const data = { __receiptImages: receiptImages.map(r => r.dataUrl), __attachedFiles: attachedFiles };
```

- [ ] **Step 3: PDF生成処理に添付ファイルの結合を追加する**

現在の1575-1593行目:
```javascript
      }
    }

        const outBytes = await a4Doc.save();
    if (mode === 'print') {
      await printViaCanvas(outBytes);
      statusEl.textContent = '印刷ダイアログを開きました';
      statusEl.className = 'status-ok';
    } else {
      const blob = new Blob([outBytes], { type: 'application/pdf' });
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      const dateStr = [v('date_year'), v('date_month'), v('date_day')].filter(Boolean).join('-') || 'output';
      a.download = `一般管理費支払伝票_${dateStr}.pdf`;
      a.click();
      URL.revokeObjectURL(url);
      statusEl.textContent = 'PDF出力完了！';
      statusEl.className = 'status-ok';
    }
```

これを以下に置き換える(既存の妙なインデント`        const outBytes`はそのまま維持し、余計な整形はしない):
```javascript
      }
    }

    // ─ 添付ファイル: PDFページを結合 ─
    const attachMergeWarnings = [];
    for (const f of attachedFiles) {
      try {
        const attachBytes = Uint8Array.from(atob(f.dataUrl.split(',')[1]), c => c.charCodeAt(0));
        const attachDoc = await PDFDocument.load(attachBytes);
        const copiedPages = await a4Doc.copyPages(attachDoc, attachDoc.getPageIndices());
        copiedPages.forEach(page => a4Doc.addPage(page));
      } catch (e) {
        attachMergeWarnings.push(f.name);
      }
    }
    const attachWarningMsg = attachMergeWarnings.length
      ? `（添付ファイルの結合に失敗: ${attachMergeWarnings.join(', ')}）`
      : '';

        const outBytes = await a4Doc.save();
    if (mode === 'print') {
      await printViaCanvas(outBytes);
      statusEl.textContent = '印刷ダイアログを開きました' + attachWarningMsg;
      statusEl.className = attachWarningMsg ? 'status-err' : 'status-ok';
    } else {
      const blob = new Blob([outBytes], { type: 'application/pdf' });
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      const dateStr = [v('date_year'), v('date_month'), v('date_day')].filter(Boolean).join('-') || 'output';
      a.download = `一般管理費支払伝票_${dateStr}.pdf`;
      a.click();
      URL.revokeObjectURL(url);
      statusEl.textContent = 'PDF出力完了！' + attachWarningMsg;
      statusEl.className = attachWarningMsg ? 'status-err' : 'status-ok';
    }
```

- [ ] **Step 4: Playwrightで動作確認する**

Task 2のStep 4と同じ手順(ダミーPDF3ページ添付→最低限の入力(事業所CD等)→PDF生成→ページ数が元の1ページ+3ページ=4ページになっていることを確認→保存/読み込み往復確認→クリア確認)を、一般管理費支払伝票アプリ_v3に対して実行する。

期待結果はTask 2と同じ。ただし出力ファイル名のプレフィックスが`一般管理費支払伝票_`になる点のみ異なる。

- [ ] **Step 5: Commit**

```bash
cd "/c/Users/shar1/OneDrive/MCフォルダ/支払伝票PJ/一般管理費支払伝票アプリ_v3"
git add "支払伝票【一般管理費】.html"
git commit -m "feat: 添付ファイルを保存/読み込み・クリアに対応し、PDF出力時に結合する"
```

---

## 全タスク完了後の後片付け

- [ ] 検証用に起動したローカルサーバー(node static-server.js)を全て停止する。
- [ ] Playwrightブラウザを閉じる(`mcp__plugin_playwright_playwright__browser_close`)。
- [ ] 検証で生成した一時PDF(`attach-test-output.pdf`等)はスクラッチパッド配下のものなので削除不要。
- [ ] pushはユーザーに確認してから実行する(このプロジェクトの既存の運用に合わせる)。
