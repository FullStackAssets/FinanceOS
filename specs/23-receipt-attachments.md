# FinanceOS Builder Spec — §23 Receipt Attachments (Items 441–460)

## Section overview
Receipt/statement attachment layer for transactions, bills, goal contributions, and recurring-template confirmations. Files are stored in-app (Retool Storage or a `file_url` link column; no external backup implied). Images get thumbnails, zoom/rotate viewing, and client-side compression; PDFs are supported view-only. Every attachment row carries `is_sample` and a SAMPLE badge when seeded.

## Tables touched
- `transactions` (existing): gains no new columns; attachments join on `transaction_id`.
- `bills` (existing, from §15): attachments join on `bill_id`.
- `goal_contributions` (existing, from §12): attachments join on `contribution_id`.
- `recurring_templates` (existing, from §17): attachments join on `template_id` + `instance_date`.
- NEW `attachments`: id (pk), transaction_id (fk, nullable), bill_id (fk, nullable), contribution_id (fk, nullable), template_id (fk, nullable), instance_date (date, nullable), file_name (text), file_url (text), file_type (text: image/jpeg, image/png, application/pdf, ...), file_size_bytes (number), width/height (number, nullable, images only), uploaded_at (timestamp), is_sample (bool). Exactly one of the four FKs is non-null per row (enforced in the insert transformer).
- `audit_log` (existing, from §38): attachment deletes and renames logged here.

## Shared cross-cutting rules
- Persistent banner "Manual tracking — no bank connection yet." appears at the top of every screen, including the receipts gallery.
- Every seeded/sample row carries `is_sample=true` with a visible SAMPLE badge; when the sample toggle hides samples, their attachments are excluded from galleries, reports, counts, and exports.
- Nothing ever presents as a live bank feed; all figures are manual entries or labeled samples.

---

### 441. Allow attaching a photo of a receipt to any transaction from iPhone camera or library.
- Components: In the transaction edit drawer, File Input `fileReceipt` with `allowMultiple=false`, accept `image/*`, and on mobile the OS presents camera vs library natively. Upload button `btnUploadReceipt`.
- Data: `qryUploadReceipt` — uploads to Retool Storage, then inserts into `attachments` (transaction_id set, is_sample inherited from the parent transaction). Transformer `trFileMeta` captures file_name, file_type, file_size_bytes before upload.
- Logic notes: Reject files over the per-file cap (see 446) before upload with an inline error. On iPhone the file input must be a real tap target ≥44px; camera capture works only over HTTPS (Retool-hosted, fine).
- Acceptance: (1) Photo taken on iPhone attaches to the correct transaction. (2) Library image attaches equally. (3) Oversize file is rejected pre-upload with a message.

### 442. Show a paperclip indicator on transactions with attachments.
- Components: In the transactions table, a custom column `colHasReceipt` rendering a 📎 icon; tooltip "1 receipt attached" (or N).
- Data: `qryAttachmentCounts` — `SELECT transaction_id, COUNT(*) AS n FROM attachments GROUP BY transaction_id`; transformer `trTxnById` maps transaction_id → count for the table's mapped data.
- Logic notes: Icon hidden when count is 0. Sample transactions show the icon plus their SAMPLE pill independently.
- Acceptance: (1) Rows with attachments show 📎; rows without show nothing. (2) Count updates immediately after upload/delete without page reload. (3) Tooltip shows the exact count.

### 443. Build a receipt viewer with zoom and rotate for attached images.
- Components: Modal `mdlReceiptViewer` with Image component `imgReceipt` (src = selected attachment file_url), zoom in/out buttons (`btnZoomIn`, `btnZoomOut`), rotate button (`btnRotate`), close button. Zoom implemented via a `zoomPct` temp state (25–400%) applied to image width style; rotate via `rotDeg` temp state (0/90/180/270) applied as CSS transform.
- Data: Selected attachment passed from the gallery or transaction drawer via `tblAttachments.selectedRow`.
- Logic notes: PDFs bypass this viewer (see 450) — the modal shows a "PDF opens in the document viewer" note with an open-original link. Zoom/rotate are view-only transforms; the stored file is unchanged.
- Acceptance: (1) Zoom 25–400% keeps the image centered and scrollable. (2) Rotate cycles 90° steps. (3) Closing resets zoom/rotation for the next open.

### 444. Support multiple attachments per transaction (receipt + warranty).
- Components: In the transaction drawer, List View `lvAttachments` bound to `qryTxnAttachments` (all rows for this transaction_id), each row showing thumbnail, file_name, file_type icon, delete button.
- Data: `qryTxnAttachments` — `SELECT * FROM attachments WHERE transaction_id = {{ drawerTxnId.value }} ORDER BY uploaded_at`. Upload appends a row; no unique constraint on (transaction_id, file_name) — duplicates allowed but flagged with "(2)" suffix in display name.
- Logic notes: The paperclip count (442) sums all rows. Deleting one row leaves the others untouched.
- Acceptance: (1) A transaction can hold receipt + warranty + photo simultaneously. (2) List shows all with correct names/types. (3) Deleting one does not affect the others.

### 445. Add attachment upload to bills for statements.
- Components: On the bills screen (§15), each bill row gets an "Attach statement" button opening `mdlBillAttachment` with a File Input accepting images + PDFs.
- Data: `qryUploadBillAttachment` — inserts into `attachments` with bill_id set (transaction_id null), is_sample inherited from the bill row.
- Logic notes: A bill may have statements for multiple months — file_name convention `billname-YYYY-MM.pdf` suggested in the modal helper text but not enforced. Bill rows show a 📎 indicator reusing the 442 pattern.
- Acceptance: (1) Statement attaches to the correct bill. (2) Bill row shows the paperclip indicator. (3) Sample bills' attachments carry is_sample=true.

### 446. Display storage usage for attachments with a per-file size cap.
- Components: Settings → Data screen: Statistic `statStorageUsed` ("Attachments: 42 MB of 500 MB"), Progress bar showing usage vs quota, Text noting the per-file cap (e.g., 10 MB).
- Data: `qryStorageUsage` — `SELECT SUM(file_size_bytes) AS total, COUNT(*) AS n FROM attachments WHERE is_sample = false OR {{ showSamples.value }}`. Constant `FILE_SIZE_CAP_MB = 10` in a temp state; enforced in the upload transformer before the Storage call.
- Logic notes: Quota is a soft in-app budget (500 MB default, editable in settings), not a Retool-enforced limit — show "over quota" warning state at 90%+. Cap applies per file; gallery thumbnails don't count extra.
- Acceptance: (1) Usage total matches the sum of stored file sizes. (2) A file over the cap is rejected pre-upload with the cap named. (3) Warning appears at ≥90% of quota.

### 447. Allow deleting an attachment with confirmation, logged in the audit log.
- Components: Delete icon per attachment row → Confirmation modal `mdlConfirmDeleteAttachment` ("Delete receipt IMG_2041.jpg? This cannot be undone.") with typed-filename confirm for files over 5 MB (extra safety), else simple Confirm/Cancel.
- Data: `qryDeleteAttachment` — deletes the `attachments` row and the Retool Storage object; on success writes `audit_log` (table=attachments, record_id, action=delete, before=file_name + file_size, actor, timestamp).
- Logic notes: Deletion is hard (no trash) — the confirm modal must say so. If the Storage delete fails, the DB row is kept and an error shown (no orphan rows).
- Acceptance: (1) Confirm deletes both file and row. (2) Cancel leaves everything intact. (3) Audit log shows the delete with file name and size.

### 448. Show thumbnails of receipts in a gallery view filterable by month.
- Components: New "Receipts" screen (under More menu): month picker `selGalleryMonth` (default current month), grid of Image components (thumbnail = file_url, CSS max 120px) bound to `qryGalleryAttachments`, click → opens the 443 viewer.
- Data: `qryGalleryAttachments` — joins attachments to transactions: `SELECT a.*, t.date, t.description, t.amount FROM attachments a JOIN transactions t ON t.id = a.transaction_id WHERE DATE_TRUNC('month', t.date) = {{ selGalleryMonth.value }}` (plus bill attachments via bill due month union). Ordered by date desc.
- Logic notes: PDFs show a PDF icon tile instead of an image thumbnail. Sample rows show SAMPLE badge on the tile and are hidden when the sample toggle is off. Empty month → friendly empty state.
- Acceptance: (1) Gallery shows only the selected month's receipts. (2) Clicking a tile opens the full viewer. (3) Sample filter toggle hides/shows sample tiles.

### 449. Tag receipts automatically with the transaction's date and amount as metadata.
- Components: No new UI — metadata shown as caption text under each gallery tile and in the viewer header: "Mar 14 · $42.50 · Whole Foods".
- Data: Set at upload time in `qryUploadReceipt`'s insert: the transformer copies `t.date` and `t.amount` from the parent transaction into the caption; stored denormalized as `caption` text on the attachments row (recomputed if the transaction is edited, via the edit-transaction success handler).
- Logic notes: Denormalized caption keeps the gallery query cheap. If the parent transaction is deleted, attachments are deleted with it (no orphan captions). Bill attachments use bill name + amount instead.
- Acceptance: (1) Every transaction attachment shows the correct date/amount caption. (2) Editing the transaction amount updates the caption. (3) Bill attachments caption from bill data.

### 450. Support PDF attachments in addition to images.
- Components: File inputs' accept attribute extended to `image/*,.pdf`; attachment rows show a PDF icon for `file_type='application/pdf'`; viewer modal (443) renders PDFs via an IFrame component `ifrPdfViewer` (src = file_url) instead of the Image component.
- Data: `file_type` column stores the MIME type; `qryUploadReceipt` allows pdf; thumbnail generation skipped for PDFs (icon tile in gallery).
- Logic notes: PDFs skip compression (455) and zoom/rotate (443) — the IFrame gives native browser PDF controls. Per-file cap (446) applies equally.
- Acceptance: (1) PDF uploads and lists correctly. (2) Gallery shows a PDF icon tile. (3) Clicking opens the PDF in the IFrame viewer.

### 451. Add a "receipts missing" report listing transactions over $50 without attachments.
- Components: Reports screen: "Receipts missing" card with table `tblMissingReceipts` (date, description, amount, account, "Attach" quick button opening the transaction drawer at the upload step).
- Data: `qryMissingReceipts` — `SELECT t.* FROM transactions t LEFT JOIN attachments a ON a.transaction_id = t.id WHERE a.id IS NULL AND ABS(t.amount) > 50 AND t.date >= {{ reportStart.value }} AND (t.is_sample = false OR {{ showSamples.value }}) ORDER BY ABS(t.amount) DESC`.
- Logic notes: Threshold $50 is a settings-editable constant (`RECEIPT_THRESHOLD`). Income transactions included (proof of deposit). Sample toggle respected.
- Acceptance: (1) Lists only >$50 transactions with zero attachments. (2) Attach-button flow removes the row from the report on success. (3) Threshold change re-runs the report.

### 452. Allow attaching receipts to goal contributions as proof of deposit.
- Components: In the goal detail contribution list (§12), each contribution row gets an attach button opening `mdlContributionAttachment` (same pattern as 441).
- Data: `qryUploadContributionAttachment` — inserts into `attachments` with contribution_id set, is_sample inherited from the contribution/goal.
- Logic notes: Contribution rows show the 📎 indicator (442 pattern). Deleting a contribution deletes its attachments (cascade in the delete-contribution handler).
- Acceptance: (1) Deposit slip photo attaches to the correct contribution. (2) Indicator shows on the contribution row. (3) Sample contributions' attachments are sample-flagged.

### 453. Show attachment count badges on the transactions table rows.
- Components: Badge component in the transactions table's attachment column showing the integer count (e.g., "3") next to the 📎 icon; hidden when 0.
- Data: Reuses `qryAttachmentCounts` from 442; transformer formats count as a small pill.
- Logic notes: Badge and icon are one combined cell renderer. Count includes images + PDFs. Updates live on upload/delete (same refresh trigger as 442).
- Acceptance: (1) "2" shows for two attachments, hidden for zero. (2) Count increments immediately after a second upload. (3) Badge respects the sample toggle (counts only visible attachments).

### 454. Add drag-and-drop receipt upload on desktop.
- Components: On desktop widths, the transaction drawer's upload zone becomes a Dropzone-style area (`cntDropzone` container with dashed border, "Drop receipts here or click to browse"); the existing File Input remains as the click target inside it.
- Data: Retool's File Input fires the same change event for dropped files — `qryUploadReceipt` is reused unchanged; a `dropActive` temp state toggles the highlight style on dragover/dragleave via event handlers.
- Logic notes: Desktop-only enhancement — hidden below 768px where the native camera/library picker (441) is the path. Multiple dropped files queue sequential uploads (444).
- Acceptance: (1) Dropping an image uploads it to the open transaction. (2) Dropzone highlights on dragover. (3) On mobile the dropzone is hidden and the standard picker works.

### 455. Compress images on upload to keep storage lean while staying readable.
- Components: No UI; a note in Settings → Data: "Images are compressed on upload (max 1600px, ~80% quality)."
- Data: In `qryUploadReceipt`'s pre-upload transformer, run client-side compression via a JS query using an offscreen canvas: downscale to max 1600px on the long edge, JPEG quality 0.8, then upload the resulting blob; store final `file_size_bytes` and dimensions post-compression.
- Logic notes: PDFs and files already under 500 KB skip compression. Compression is lossy but receipt-legible — never applied to warranty documents the user flags "keep original" via a checkbox on the upload modal.
- Acceptance: (1) A 4 MB phone photo stores at well under 1 MB. (2) Text on the receipt remains legible in the viewer. (3) "Keep original" checkbox bypasses compression.

### 456. Display a warning that attachments are stored in-app and not backed up externally.
- Components: Warning Callout `callBackupWarning` on the Receipts gallery screen and in the upload modal: "Receipts are stored inside FinanceOS only — they are not backed up externally. Export important documents elsewhere."
- Data: Static text; links to the CSV export (460 includes attachment presence) and the full JSON export (§39/§50 data-management screen).
- Logic notes: Shown once per session as a dismissible callout on the gallery; always visible (non-dismissible) inside the upload modal. Plain language, no jargon.
- Acceptance: (1) Warning visible on the gallery until dismissed, restored next session. (2) Warning always present in the upload modal. (3) Copy mentions the export path.

### 457. Support viewing attachments offline once downloaded.
- Components: In the viewer modal, "Save for offline" button `btnSaveOffline` per attachment; gallery tiles show a small downloaded checkmark when cached.
- Data: Uses the browser Cache API via a JS query `jsCacheAttachment` (fetch file_url → cache.put); viewer first tries `caches.match` before network. A `offlineReady` flag per attachment kept in localStorage (key = attachment id).
- Logic notes: Best-effort — works only in the same browser/profile; the 456 warning still applies (cache is not a backup). PDFs cache the same way. If the cached copy is missing, fall back to network with a note.
- Acceptance: (1) "Save for offline" then airplane mode → image still opens. (2) Cached tiles show the checkmark. (3) Uncached attachments show a clear offline error, not a blank viewer.

### 458. Add receipt attachment to recurring-template confirmations.
- Components: In the confirmation queue (§17/§323), each pending instance row gets an "Attach receipt" button; the confirm dialog includes an optional file input.
- Data: `qryUploadInstanceAttachment` — inserts into `attachments` with template_id + instance_date set; when the instance is confirmed into a real transaction, a follow-up update re-points the row to the new transaction_id (keeping template_id for traceability).
- Logic notes: Attachment is optional — confirming without one works normally. If the instance is skipped, its draft attachment is deleted with it.
- Acceptance: (1) Receipt attaches during confirmation. (2) After confirm, the attachment appears on the created transaction. (3) Skipping the instance removes its draft attachment.

### 459. Allow renaming attachment files for organization.
- Components: Edit (pencil) icon per attachment row → inline Text Input prefilled with current file_name → Save/Cancel.
- Data: `qryRenameAttachment` — `UPDATE attachments SET file_name = {{ newName.value }} WHERE id = ...`; writes audit_log (action=rename, before/after file_name).
- Logic notes: Extension is preserved automatically (rename edits the base name only; the input shows the name without extension). Duplicate names allowed (444). Empty name rejected.
- Acceptance: (1) Rename updates everywhere (list, gallery caption, viewer header). (2) Extension preserved. (3) Audit log records old → new name.

### 460. Include attachment presence in CSV exports as a true/false column.
- Components: No UI change — the transactions CSV export (item 481) gains a `has_attachment` column; the export-column picker (491) lists it as a toggleable column defaulting to ON.
- Data: Export query LEFT JOINs the attachment counts: `..., (COUNT(a.id) > 0) AS has_attachment ... GROUP BY t.id`; renders as TRUE/FALSE strings in the CSV.
- Logic notes: Counts only attachments visible under the current sample toggle (hidden samples → FALSE). Bills/goal exports (484/492) get the analogous column for their attachment types.
- Acceptance: (1) CSV has a has_attachment column with TRUE/FALSE per row. (2) A transaction with 2 receipts exports TRUE (not the count). (3) Column respects the sample toggle and the export column picker.
