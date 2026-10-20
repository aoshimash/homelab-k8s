# Document Ingest - ScanSnap PDFs to Outline

Scanned paperwork lands in Google Drive and is filed and indexed automatically:
a scheduled Claude Code task reads each new PDF, files it into the Drive folder
tree under a descriptive name, and creates a searchable Outline document that
links back to the original. It replaces paperless-ngx's consume-and-OCR
pipeline. Introduced by
[#379](https://github.com/aoshimash/homelab-k8s/issues/379), part of
[#376](https://github.com/aoshimash/homelab-k8s/issues/376).

**No code for this pipeline lives in this repository.** It is a prompt run on a
schedule. This page is its contract: what a finished document looks like, how
files are filed and named, how processed files are recognized, and the exact
prompt, so the setup can be recreated or changed without guesswork.

## Overview

```
ScanSnap ──> Google Drive "00_受信箱"
                 │  daily, scheduled Claude Code task on the owner's Mac
                 ├──> Outline collection "書類" (summary + transcription + Drive link)
                 └──> Drive: renamed and moved into the category folder tree
```

| Item | Value |
|------|-------|
| Runtime | Scheduled task `document-ingest` in the Claude desktop app (Code), on the owner's Mac |
| Schedule | Daily at 21:00 local time (`0 21 * * *`) |
| Input | PDFs directly in Drive folder `00_受信箱` (ID `11_RuOPedK3r8koOu519nULsmTR5WDCFu`, My Drive root) |
| Drive access | Claude's Google Drive connector (`claude.ai Google Drive`) for listing, renaming and moving |
| Reading PDFs | The local copy synced by Google Drive for desktop, rendered to page images with `pdftoppm` (poppler), read by the model |
| Cross-check | macOS Vision OCR of the same pages; values the OCR does not contain are re-checked and marked |
| Outline access | MCP server `outline` (`https://outline.tail19032f.ts.net/mcp`, see [outline.md](outline.md#connecting-claude-code-over-mcp)) |
| Output | Outline collection `書類`; the PDF renamed and moved within Drive |
| Max files per run | 20 |

Runs need the Mac awake, the desktop app open, and the Mac on the tailnet. A
missed run is harmless: the next run processes whatever is still in the inbox.
The desktop app runs a task that was due while it was closed on its next
launch.

PDF contents are sent to Claude for reading. That privacy trade-off was
accepted in [#376](https://github.com/aoshimash/homelab-k8s/issues/376).

### Why PDFs are read from the synced copy

The Google Drive connector returns a PDF either as text from Google's own OCR
or as a base64 string. The OCR text is incomplete for many scans (photos,
small print, tables). The base64 of even a 1 MB PDF is far too large for the
model to read as an image. The first runs on 2026-10-20 went this way and
produced two transcriptions with missing parts and one guessed amount.

Google Drive for desktop already syncs My Drive to
`~/Library/CloudStorage/GoogleDrive-aoshimash@gmail.com/マイドライブ/`. Every
synced file carries its Drive file ID in the extended attribute
`com.google.drivefs.item-id#S`. So the task finds the local file by ID, renders
each page with `pdftoppm -r 150 -png`, and reads the page images. The images
are the source of truth; the connector's OCR text is only a hint. The page
images contain unmasked data, so they go in a fresh temporary directory that
is deleted as soon as the file is done.

A PDF that is not in the synced folder yet (sync lag) is skipped for that run
and reported as `未同期`. It is still in the inbox, so the next run picks it up.

### Why the model reads, and OCR only cross-checks

On 2026-10-20, three methods were compared on 7 pages from 3 documents: a
motorcycle-school price list, a scholarship repayment notice, and a driver's
licence renewal notice. They were scored on 70 values (amounts, dates,
deadlines, business phone numbers) checked by hand at 300 dpi. Personal
identifiers were left out of the scoring.

| Method | Values found | Time | Tables | Flags doubt |
|--------|-------------:|------|--------|-------------|
| Model (Sonnet) reading 150 dpi page images | 70/70 | 27–44 s per document | Kept: 9 × 12 price table correct cell by cell | Yes |
| macOS Vision OCR, 300 dpi | 68/70 | ~1–2 s per page (first call ~30 s) | Lost (flat lines) | No |
| Tesseract 5.5 `jpn+eng`, 300 dpi | 47/70 | ~2–6 s per page | Lost | No |

The model is the only method that keeps tables and can say it is unsure, so it
writes the page. Vision OCR is cheap, reads digits reliably, and fails in
different ways from the model. So each value the model writes is looked up in
the OCR text. A value the OCR does not contain is re-read from the image and
either corrected or marked `（要確認: OCR不一致）`. This catches confident
misreadings, the failure that doubt-flagging alone cannot catch. Tesseract
missed most amounts and both deadline dates on the licence notice, and is not
used.

## Contract

### Filing in Drive

- **Destination.** A subfolder of one of the numbered category folders at the
  My Drive root (`01_仕事` … `07_生活`), chosen from the document's content.
  If no existing subfolder fits, the task creates one under the best-fitting
  category folder, named like the existing ones: a short Japanese noun such as
  `税務` or `保険`. A document that fits a category folder but no specific
  topic may go directly into that category folder.
- **Never used as destinations:** `00_受信箱` itself, `98_共有`, `99_アーカイブ`,
  and folders that are not numbered categories (e.g. `Google Meet`). New
  top-level folders are never created.
- **File name:** `YYYYMMDD_書類名.pdf`. `YYYYMMDD` is the document's own date
  (issue date) when it can be read. Otherwise it is the scan date, which is
  the file's Drive `createdTime` in JST. `書類名` is a short descriptive title,
  e.g. `20261004_運転免許証更新のお知らせ.pdf`. `/` and other characters
  invalid in names are replaced with `_`.
- **Undecidable files stay in the inbox.** If the destination cannot be decided
  with confidence (an unreadable scan, or several unrelated documents in one
  PDF), the file is left in `00_受信箱` under its original name. It is still
  registered in Outline, with `保管場所` set to `要振り分け`, and it is listed in
  the run report. The owner files it by hand.

### Placement in Outline

- Collection **`書類`**, created by the task if missing.
- Documents are nested under parent documents that mirror the Drive path:
  `書類 / 06_諸手続き / 税務 / 20261004_…`. A parent document's title is exactly
  the folder name, and its body is a link to the Drive folder. Missing parents
  are created.
- Files left as `要振り分け` go under a parent document `00_受信箱（要振り分け）`.
- If the Outline MCP tools cannot nest documents, documents go directly in the
  collection, and the run report says so.

### Document format

Title: the same as the file name without `.pdf` (`20261004_運転免許証更新のお知らせ`).
Body, in Japanese:

```markdown
> このページは自動取り込みで作成されました。機微な番号は伏せ字にしています。原本は下の Drive リンクを参照してください。

| 項目 | 値 |
|------|-----|
| 書類日付 | 2026-10-04（読み取れずスキャン日を使った場合は「2026-10-04（スキャン日）」） |
| 書類名 | 運転免許証更新のお知らせ |
| 種別 | 通知 / 請求書 / 領収書 / 契約書 / 証明書 / 明細 / 申請書 / その他 |
| 発行元 | 〇〇県公安委員会 |
| 期限・金額 | 期限・支払額・有効期間などがあれば。なければ「なし」 |
| 要確認 | 判読に自信のない項目の一覧。なければ「なし」 |
| 原本 | [20261004_運転免許証更新のお知らせ.pdf](https://drive.google.com/file/d/<FILE_ID>/view) |
| 保管場所 | Drive: 06_諸手続き/身分（または「要振り分け」） |
| Drive ファイル ID | `<FILE_ID>` |
| スキャン日時 | Drive createdTime（JST） |
| 取り込み日時 | 実行日時（JST） |

## 要約

2〜4 文。何の書類か、誰から、必要な対応と期限。

## 本文（書き起こし）

PDF の全文を Markdown で書き起こしたもの。見出し・箇条書き・表は構造を保つ。複数ページは `### p.2` のように区切る。
```

**Sensitive numbers are masked** in every part of the page, with only the last
4 digits kept (`****1234`). This covers My Number (マイナンバー), full credit
card numbers, bank account numbers, passwords and PINs, and identifiers that
alone allow access or impersonation: 基礎年金番号, health insurance 記号・番号,
driver's licence and passport numbers. The original in Drive is unmasked.

**Uncertain readings are marked, never guessed.** A value that cannot be read
is written as `（判読不能）`. A value read with doubt is written followed by
`（要確認）`, e.g. `30,690円（要確認）`. Every such item is also listed in the
`要確認` row and in the run report, so the owner can go through them later with
the original. A value the model reads confidently but the OCR does not contain
is marked `（要確認: OCR不一致）` after a second look at the image.

### Processed files, retries, and duplicates

- **A file in `00_受信箱` is unprocessed** unless an Outline document already
  carries its Drive file ID. The ID never changes when a file is renamed or
  moved, so it is the key; titles are not.
- **Order per file: Outline first, then Drive.** The document is created
  before the file is renamed and moved. A run that dies in between leaves a
  file that is still in the inbox and already has a document. The next run
  finds the document by file ID and only completes the rename and move,
  reading the destination from the document's `保管場所`.
- A file whose document says `要振り分け` is skipped on later runs and keeps
  being listed in the report until the owner files it.
- A file that fails before its document is created is retried in full on the
  next run, because it is still in the inbox with no document.
- The task never deletes or trashes anything in Drive or Outline, and never
  edits Outline documents other than the ones it created (identified by the
  `Drive ファイル ID` line) and the folder parent documents.

## Scheduled Task

The task is stored by the desktop app in
`~/.claude/scheduled-tasks/document-ingest/SKILL.md`. That file is the source
of truth on the Mac; the prompt below is its recorded copy. **Update both
together.**

To recreate it, ask Claude Code in the desktop app to create a scheduled task
with task ID `document-ingest`, title `書類の取り込み（ScanSnap → Outline）`,
cron `0 21 * * *`, and exactly the prompt below. The runs need the `outline`
MCP server (user scope) and the Google Drive connector to be connected, Google
Drive for desktop syncing My Drive, `pdftoppm` (`brew install poppler`), and
the two helpers below in `~/.claude/scheduled-tasks/document-ingest/`.

### Helpers

Both live next to the task's `SKILL.md`, outside this repository; the sources
are recorded here.

`vision-ocr.swift` — build with
`swiftc -O vision-ocr.swift -o vision-ocr` (Xcode Command Line Tools):

```swift
// OCR an image with macOS Vision (VNRecognizeTextRequest) and print the text,
// one recognized line per output line, top to bottom.
// Usage: swift vision-ocr.swift <image.png>
import Foundation
import Vision
import AppKit

guard CommandLine.arguments.count == 2 else {
    FileHandle.standardError.write("usage: vision-ocr <image>\n".data(using: .utf8)!)
    exit(2)
}
let url = URL(fileURLWithPath: CommandLine.arguments[1])
guard let image = NSImage(contentsOf: url),
      let cg = image.cgImage(forProposedRect: nil, context: nil, hints: nil) else {
    FileHandle.standardError.write("cannot load image\n".data(using: .utf8)!)
    exit(1)
}
let request = VNRecognizeTextRequest()
request.recognitionLevel = .accurate
request.recognitionLanguages = ["ja-JP", "en-US"]
request.usesLanguageCorrection = true
let handler = VNImageRequestHandler(cgImage: cg, options: [:])
try handler.perform([request])
let lines = (request.results ?? [])
    .compactMap { obs -> (CGRect, String)? in
        guard let s = obs.topCandidates(1).first?.string else { return nil }
        return (obs.boundingBox, s)
    }
    // Vision's origin is bottom-left: sort top to bottom, then left to right.
    .sorted { a, b in
        abs(a.0.midY - b.0.midY) > 0.01 ? a.0.midY > b.0.midY : a.0.minX < b.0.minX
    }
for (_, s) in lines { print(s) }
```

`crosscheck.py`:

```python
"""Report which values do not appear in OCR text, after normalization.

Usage: python3 -I crosscheck.py <ocr.txt> <value> [<value> ...]

Prints one line per value that is NOT found, and nothing for values that are.
Normalization: Unicode NFKC (full-width digits and symbols to ASCII), all
whitespace removed, and the various wave dashes and minus signs unified.
"""
import re
import sys
import unicodedata


def norm(s: str) -> str:
    s = unicodedata.normalize("NFKC", s)
    s = re.sub(r"[〜～~]", "~", s)
    s = re.sub(r"[‐‑‒–—―−ｰ]", "-", s)
    return re.sub(r"\s+", "", s)


def main() -> int:
    if len(sys.argv) < 3:
        print("usage: crosscheck.py <ocr.txt> <value> [<value> ...]", file=sys.stderr)
        return 2
    text = norm(open(sys.argv[1], encoding="utf-8").read())
    for value in sys.argv[2:]:
        if norm(value) not in text:
            print(value)
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

Prompt (version 3, 2026-10-20: reads page images and cross-checks values against macOS Vision OCR; version 2 read page images only; version 1 read the connector's OCR text):

````text
You are the document-ingest job for the owner's scanned paperwork. Work fully autonomously; nobody is watching. Write everything you create (Outline pages, folder names, the final report) in Japanese.

Tools: the Google Drive connector (claude.ai Google Drive) for listing, renaming and moving files; the Outline MCP server named "outline"; Bash and Read for reading PDFs from the local Drive sync. If the Drive connector or Outline is unavailable, or `pdftoppm` is not installed, stop immediately and report which one.

Helpers in HELPERS="$HOME/.claude/scheduled-tasks/document-ingest": `vision-ocr` (macOS Vision OCR of one image, prints text) and `crosscheck.py` (prints the given values that do not occur in an OCR text file). If `vision-ocr` is missing, build it once with: swiftc -O "$HELPERS/vision-ocr.swift" -o "$HELPERS/vision-ocr". If it still cannot run, continue without the cross-check and say so in the report.

Constants:
- INBOX: Drive folder "00_受信箱", ID 11_RuOPedK3r8koOu519nULsmTR5WDCFu (in My Drive root).
- DRIVE_ROOT (local sync by Google Drive for desktop): "$HOME/Library/CloudStorage/GoogleDrive-aoshimash@gmail.com/マイドライブ". Each synced file has its Drive file ID in the extended attribute "com.google.drivefs.item-id#S".
- CATEGORY FOLDERS: the folders directly in My Drive root whose names start with 01_ to 07_ (e.g. 01_仕事, 06_諸手続き, 07_生活). Never use 00_受信箱, 98_共有, 99_アーカイブ, or any non-numbered folder as a destination. Never create a new top-level folder.
- OUTLINE COLLECTION: "書類" (create it if missing).
- MAX_FILES: 20 (process at most this many files per run; the rest wait for the next run).

Procedure:
1. List the PDFs whose parent is INBOX (mimeType application/pdf). Ignore other files, but mention them in the report. Process the oldest first (by createdTime).
2. List the CATEGORY FOLDERS and their subfolders (two levels), so you know the filing tree.
3. For each PDF (up to MAX_FILES):
   a. Duplicate check: search the Outline collection "書類" for the file's Drive ID. If a document created by this job contains "Drive ファイル ID" with this ID:
      - If its 保管場所 is "要振り分け": skip the file (list it in the report as still needing manual filing).
      - Otherwise: the document exists but the file was not moved. Rename and move the file as the document's title and 保管場所 say (step g), and do nothing else.
   b. Read the PDF from its page images. Never read it only from text.
      - Find the local copy: in "$DRIVE_ROOT/00_受信箱", the *.pdf whose `xattr -p 'com.google.drivefs.item-id#S'` equals the Drive file ID. If none matches (not synced yet), skip this file, report it as "未同期", and continue; it is retried next run.
      - TMP=$(mktemp -d); pdftoppm -r 150 -png "<local file>" "$TMP/p"; then open every page image with the Read tool, in page order. If the PDF has more than 30 pages, read the first 30 and say so in the page and the report.
      - Also run OCR for the cross-check: pdftoppm -r 300 -png "<local file>" "$TMP/o"; then for each "$TMP"/o-*.png in page order run "$HELPERS/vision-ocr" on it and append the output to "$TMP/ocr.txt". Do not read ocr.txt as the transcription; it is only for step c's cross-check.
      - The images are the source of truth. You may also call read_file_content on the Drive file as a hint, but never copy text from it that you cannot see in the images.
      - When this file is finished (success or failure), run rm -rf "$TMP". The images contain unmasked personal data.
   c. Extract: 書類日付 (the document's issue date; if none is printed, use the file's createdTime converted to JST and mark it "（スキャン日）"), 書類名 (short descriptive title), 種別 (通知/請求書/領収書/契約書/証明書/明細/申請書/その他), 発行元, 期限・金額 (deadlines, amounts, validity periods; "なし" if none), 要約 (2–4 sentences: what it is, from whom, required action and deadline), and a full Markdown transcription that keeps headings, lists and tables, with pages separated by "### p.N".
      Uncertain readings: write "（判読不能）" for anything you cannot read, and put "（要確認）" right after any value you read with doubt (e.g. "30,690円（要確認）"). Never present a guess as fact. Collect every such item for the 要確認 row.
      OCR cross-check: run python3 -I "$HELPERS/crosscheck.py" "$TMP/ocr.txt" followed by every value you read for the 書類日付 and 期限・金額 rows and every amount, date, deadline, time, phone number and reference number in your transcription, each passed exactly as printed on the page (not reformatted, and before masking). Skip a 書類日付 marked "（スキャン日）". For each value it prints (not found in the OCR text), look at that spot in the page image again: if you misread it, correct it; if you still read it the same way, append "（要確認: OCR不一致）" to it and add it to the 要確認 row.
   d. Mask sensitive numbers everywhere you write them, keeping only the last 4 digits as "****1234": My Number (マイナンバー), full credit card numbers, bank account numbers, passwords and PINs, 基礎年金番号, health insurance 記号・番号, driver's licence numbers, passport numbers.
   e. Decide the destination: the best-fitting existing subfolder of a CATEGORY FOLDER. If none fits, you may create one new subfolder under the best-fitting CATEGORY FOLDER, named like the existing ones (a short Japanese noun such as 税務, 保険). A document that fits a category but no specific topic may go directly in the category folder. If you cannot decide with confidence (unreadable scan, several unrelated documents in one PDF), the destination is "要振り分け": the file stays in INBOX with its original name.
      Title = "YYYYMMDD_書類名" using 書類日付 (replace "/" and other characters invalid in names with "_"). File name = title + ".pdf".
   f. Create the Outline document FIRST, in collection "書類", nested under parent documents that mirror the Drive path (e.g. 書類 / 06_諸手続き / 税務). A parent document's title is exactly the folder name and its body is a link to that Drive folder; create missing parents. Put "要振り分け" documents under a parent titled "00_受信箱（要振り分け）". If the Outline tools cannot nest documents, create it directly in the collection and say so in the report.
      Document title: the title from step e. Body, exactly in this shape:
        > このページは自動取り込みで作成されました。機微な番号は伏せ字にしています。原本は下の Drive リンクを参照してください。
        a two-column table "| 項目 | 値 |" with rows: 書類日付, 書類名, 種別, 発行元, 期限・金額, 要確認 (the uncertain items, or "なし"), 原本 (a Markdown link whose text is the new file name and whose URL is https://drive.google.com/file/d/<FILE_ID>/view), 保管場所 ("Drive: <category>/<subfolder>" or "要振り分け"), Drive ファイル ID (the ID in backticks), スキャン日時 (createdTime in JST), 取り込み日時 (now, JST)
        "## 要約" followed by the summary
        "## 本文（書き起こし）" followed by the transcription
   g. Only after the document exists: if the destination is not "要振り分け", rename the Drive file to the new file name and move it into the destination folder (one update that sets title and parentId). Then confirm the file's parent is the destination.
   h. If any step fails for a file, record the error and continue with the next file. A file whose document was not created stays in INBOX and is retried in full next run.
4. Safety rules: never delete or trash anything in Drive or Outline (the only thing you delete is your own $TMP directory). Never touch Drive files other than the one being filed. Never edit Outline documents except ones this job created (they contain "Drive ファイル ID") and the folder parent documents.
5. Final report (your last message), in Japanese: a table with one row per processed file — 元のファイル名, 新しいファイル名, 振り分け先, Outline のリンク, 要確認 (count), OCR照合 (values checked / corrected / OCR不一致), 結果 (登録・移動済み / 移動のみ再実行 / 要振り分け / 未同期 / エラー: reason). Then list: every 要確認 item per file, new Drive folders created, files skipped because of MAX_FILES, files still needing manual filing, and non-PDF files in INBOX. If INBOX had nothing to do, say so in one line.
````

## Operations

### Check recent runs

In the desktop app, open the routine **書類の取り込み（ScanSnap → Outline）**.
Each run's last message is the report described above.

### A document was filed wrongly

1. Move the file in Drive to the right folder by hand.
2. In Outline, update `保管場所` in the document's table and move the document
   under the matching parent document.

The task does not touch a file once it has left `00_受信箱`.

### Re-ingest documents that are already filed

To re-read files whose Outline documents already exist (after a contract
change, or when a page looks wrong), run the prompt below once as an ad-hoc
Claude Code task, with `FILE_IDS` filled in from the documents'
`Drive ファイル ID` rows. It updates each document in place, so nothing is
duplicated, and it renames or moves the Drive file only if the new reading
changes its title or folder.

On 2026-10-20 this re-read the eight documents created by prompt version 1,
first from page images (version 2) and then again with the OCR cross-check
(version 3).

````text
You are re-ingesting documents for the owner's document-ingest pipeline. Work fully autonomously. Write everything you create (Outline pages, the final report) in Japanese. The contract is the same as the document-ingest job; this run re-reads files that were already ingested and updates their existing Outline documents in place.

Tools: the Google Drive connector (claude.ai Google Drive); the Outline MCP server named "outline"; Bash and Read. Stop and report if any is unavailable or `pdftoppm` is missing.

Constants:
- DRIVE_ROOT (local sync by Google Drive for desktop): "$HOME/Library/CloudStorage/GoogleDrive-aoshimash@gmail.com/マイドライブ". Each synced file has its Drive file ID in the extended attribute "com.google.drivefs.item-id#S".
- OUTLINE COLLECTION: "書類".
- FILE_IDS: <one Drive file ID per line>

For each ID in FILE_IDS:
1. Find its Outline document: the one in "書類" whose "Drive ファイル ID" row is this ID. If there is none, or more than one, skip the ID and report it. Never create a new document in this run.
2. Get the file's Drive metadata (title, parent folder) and resolve its folder path. Find the local copy under DRIVE_ROOT in that folder whose `xattr -p 'com.google.drivefs.item-id#S'` equals the ID; if it is not there, search DRIVE_ROOT with `find` (max depth 4) for a PDF with that attribute. If no local copy exists, skip and report "未同期".
3. TMP=$(mktemp -d); pdftoppm -r 150 -png "<local file>" "$TMP/p"; open every page image with the Read tool in page order (first 30 pages at most). The images are the source of truth. Also run pdftoppm -r 300 -png "<local file>" "$TMP/o" and append the output of "$HOME/.claude/scheduled-tasks/document-ingest/vision-ocr" for each "$TMP"/o-*.png, in page order, to "$TMP/ocr.txt". Afterwards always rm -rf "$TMP": the images contain unmasked personal data.
4. Re-extract everything exactly as the document-ingest contract says: 書類日付 (issue date, or createdTime in JST marked "（スキャン日）"), 書類名, 種別, 発行元, 期限・金額, 要約, and a full Markdown transcription with "### p.N" page breaks. Write "（判読不能）" for anything unreadable and put "（要確認）" after any value read with doubt; list them all in the 要確認 row. Then cross-check: run python3 -I "$HOME/.claude/scheduled-tasks/document-ingest/crosscheck.py" "$TMP/ocr.txt" with every value for 書類日付 and 期限・金額 and every amount, date, deadline, time, phone number and reference number in the transcription, exactly as printed and before masking (skip a 書類日付 marked "（スキャン日）"). For each value it prints, look at the image again: correct a misreading, or append "（要確認: OCR不一致）" and add it to the 要確認 row. Mask sensitive numbers to their last 4 digits ("****1234"): My Number, full card numbers, bank account numbers, passwords and PINs, 基礎年金番号, health insurance 記号・番号, driver's licence numbers, passport numbers.
5. Keep the current Drive folder unless the new reading makes it clearly wrong; do not create folders in this run.
6. Update the existing Outline document in place: title "YYYYMMDD_書類名", and a body in exactly the contract's shape (the blockquote; the table with rows 書類日付, 書類名, 種別, 発行元, 期限・金額, 要確認, 原本, 保管場所, Drive ファイル ID, スキャン日時, 取り込み日時 = now in JST; "## 要約"; "## 本文（書き起こし）"). If the title changed, rename the Drive file to match (title + ".pdf"). If the folder changed, move both the Drive file and the Outline document (under the parent documents that mirror the folder path).
7. Never delete or trash anything except your own $TMP directory. Never touch other files or documents.

Final report (Japanese): a table with one row per ID — 新しいタイトル, 振り分け先, Outline のリンク, 要確認 (count), OCR照合 (values checked / corrected / OCR不一致), 結果. Then, per file, what changed in substance compared with the previous version of the page (dates, amounts, deadlines, title, folder), and every 要確認 item.
````

### Re-ingest a file from scratch

Delete (or archive) its Outline document, and move the PDF back into
`00_受信箱`. The next run treats it as new.

### Change the contract

Edit this page and the task prompt in the same change: update the scheduled
task (`~/.claude/scheduled-tasks/document-ingest/SKILL.md`, or ask Claude Code
to update the task), then the prompt copy above. Documents created earlier
keep their old shape.

## References

- [outline.md](outline.md) — the Outline deployment and its MCP endpoint
- [#376](https://github.com/aoshimash/homelab-k8s/issues/376) — knowledge hub initiative, decisions on runtime and original storage
- [#379](https://github.com/aoshimash/homelab-k8s/issues/379) — this pipeline
- [#380](https://github.com/aoshimash/homelab-k8s/issues/380) — migrating paperless-ngx documents through this pipeline
