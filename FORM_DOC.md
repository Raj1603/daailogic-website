# Ideal Avatar Form — Submission & Apps Script Capture

## Overview

This document explains how the `ideal-avatar.html` form submits data, how the Google Apps Script webhook (`doPost`) captures and writes submissions into a Google Sheet, and how to deploy and test the full flow.

Files involved

- [ideal-avatar.html](ideal-avatar.html) — client form page that builds a labeled payload and appends `_field_order` using `|||` as the separator.
- [FORM_DOC.md](FORM_DOC.md) — this documentation.

## Client-side payload format

- The client builds fields where the key is the visible question label text (label[for=id].innerText) and the value is the field value.
- Hidden control fields (e.g., `_subject`) are included as submitted.
- The client appends a helper field named `_field_order` with value `Label 1|||Label 2|||Label 3...` so the webhook can preserve the exact question order.

Example (urlencoded form payload):

- `Name=John+Doe&Email=john%40example.com&What+is+your+goal%3F=Build+income&_field_order=Name|||Email|||What+is+your+goal%3F`

Notes

- The client uses `FormData()` when sending; Apps Script will receive the values in `e.parameter` or via `e.postData.contents` if urlencoded.

## Apps Script `doPost` (paste into Apps Script project)

Paste the following into your Apps Script project and replace the three constants before deployment.

```javascript
// Replace these constants before deploying
const SHEET_ID = 'REPLACE_WITH_YOUR_SHEET_ID';
const SHEET_NAME = 'Sheet1'; // or your sheet name
const EMAIL_TO = 'you@example.com';

function parseQueryString(qs) {
  const obj = {};
  qs.split('&').forEach(pair => {
    if (!pair) return;
    const parts = pair.split('=');
    const k = decodeURIComponent(parts.shift().replace(/\+/g, ' '));
    const v = decodeURIComponent((parts.join('=') || '').replace(/\+/g, ' '));
    if (obj[k] === undefined) obj[k] = v;
    else if (Array.isArray(obj[k])) obj[k].push(v);
    else obj[k] = [obj[k], v];
  });
  return obj;
}

function normalizeValue(v) {
  if (v === null || v === undefined) return '';
  if (Array.isArray(v)) return v.join(', ');
  return String(v);
}

function doPost(e) {
  try {
    // 1) Build a flat object `data` from the incoming request
    let data = {};
    if (e.postData && e.postData.type && e.postData.type.indexOf('application/json') === 0) {
      try { data = JSON.parse(e.postData.contents || '{}'); } catch (err) { data = {}; }
    } else if (e.parameter && Object.keys(e.parameter).length) {
      for (let k in e.parameter) { const val = e.parameter[k]; data[k] = Array.isArray(val) ? val[0] : val; }
    } else if (e.postData && e.postData.contents) {
      data = parseQueryString(e.postData.contents);
    }

    // 2) Extract and remove the helper `_field_order`
    let fieldOrder = [];
    if (data._field_order) {
      fieldOrder = String(data._field_order).split('|||').map(s => s.trim()).filter(Boolean);
      delete data._field_order;
    } else {
      fieldOrder = Object.keys(data).slice();
    }

    // 3) Build header row and row values (Timestamp first)
    const headers = ['Timestamp', ...fieldOrder];
    const row = [new Date().toISOString()];
    for (let key of fieldOrder) row.push(normalizeValue(data[key]));

    // 4) Open sheet and ensure header row matches `headers`
    const ss = SpreadsheetApp.openById(SHEET_ID);
    const sheet = ss.getSheetByName(SHEET_NAME) || ss.getSheets()[0];

    if (sheet.getLastColumn() < headers.length) {
      sheet.insertColumnsAfter(sheet.getLastColumn() || 1, headers.length - sheet.getLastColumn());
    }
    sheet.getRange(1, 1, 1, headers.length).setValues([headers]);

    // 5) Append the new row
    sheet.appendRow(row);

    // 6) Send notification email with fields in the same order
    const lines = headers.map((h, i) => `${h}:\n${row[i] || ''}`);
    const body = lines.join('\n\n');

    MailApp.sendEmail({ to: EMAIL_TO, subject: 'New Ideal Avatar Submission', body: body });

    return ContentService.createTextOutput(JSON.stringify({ status: 'success' }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService.createTextOutput(JSON.stringify({ status: 'error', message: err.message }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}
```

### Replace placeholders

- `SHEET_ID` — the ID from the Google Sheet URL.
- `SHEET_NAME` — the sheet/tab name to write into (default `Sheet1`).
- `EMAIL_TO` — the notification recipient email.

## Deploy as Web App

1. Save the script.
2. Click **Deploy → New deployment**.
3. Choose **Web app**.
4. Set **Execute as**: `Me`.
5. Set **Who has access**: `Anyone` or `Anyone, even anonymous` (required for public static sites to POST without auth).
6. Deploy and copy the Web App URL.

## Update the client (`ideal-avatar.html`)

Open [ideal-avatar.html](ideal-avatar.html) and replace the placeholder `appsScriptUrl` value with the deployed Web App URL.

Example location in the file:

```js
const appsScriptUrl = 'https://script.google.com/macros/s/XXXXX/exec'; // replace
```

## Testing

From a shell using urlencoded data (simulates the client FormData POST):

curl example (Linux/macOS):

```bash
curl -X POST 'https://your-deployed-url/exec' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data 'Name=Tester&Email=test%40example.com&_field_order=Name|||Email'
```

PowerShell example:

```powershell
Invoke-RestMethod -Uri 'https://your-deployed-url/exec' -Method Post -Body @{ Name='Tester'; Email='test@example.com'; _field_order='Name|||Email' } -ContentType 'application/x-www-form-urlencoded'
```

After a successful POST:

- The Google Sheet will have its first row overwritten with `Timestamp` + the label order from `_field_order`.
- A notification email will be sent to `EMAIL_TO` with values ordered the same way.

## Troubleshooting

- `_field_order` appears as a column in the sheet or email: you are running an older Apps Script that does not delete `_field_order`. Redeploy the script above and ensure the deployed URL matches the client.
- Empty fields in the sheet: confirm the client is sending values and that label text matches what's expected; check the FormData send behavior in the browser DevTools Network tab.
- Web app returns 403/401: ensure deployment access is `Anyone` and you re-deployed after edits.
- Apps Script timeout: keep payloads small; Apps Script has execution time limits.

## Security & Notes

- Deploying as `Anyone` allows unauthenticated POSTs; ensure you do not accept or store sensitive information (passwords, credit cards) in plain text.
- Consider adding a simple token field (hidden input) and verifying it in `doPost` if you want a minimal authenticity check (not cryptographically secure, but reduces stray spam).

---

If you want, I can also: (A) fill `SHEET_ID`, `SHEET_NAME`, and `EMAIL_TO` in the script for you; (B) replace the `appsScriptUrl` placeholder in `ideal-avatar.html` with your deployed URL; (C) run a test submission from this environment.
