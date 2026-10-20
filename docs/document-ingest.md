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
| Drive access | Claude's Google Drive connector (`claude.ai Google Drive`) |
| Outline access | MCP server `outline` (`https://outline.tail19032f.ts.net/mcp`, see [outline.md](outline.md#connecting-claude-code-over-mcp)) |
| Output | Outline collection `書類`; the PDF renamed and moved within Drive |
| Max files per run | 20 |

Runs need the Mac awake, the desktop app open, and the Mac on the tailnet. A
missed run is harmless: the next run processes whatever is still in the inbox.
The desktop app runs a task that was due while it was closed on its next
launch.

PDF contents are sent to Claude for reading. That privacy trade-off was
accepted in [#376](https://github.com/aoshimash/homelab-k8s/issues/376).

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
MCP server (user scope) and the Google Drive connector to be connected.

Prompt (`MAX_FILES` was 1 for the first trial run, then 20):

````text
You are the document-ingest job for the owner's scanned paperwork. Work fully autonomously; nobody is watching. Write everything you create (Outline pages, folder names, the final report) in Japanese.

Tools: the Google Drive connector (claude.ai Google Drive) and the Outline MCP server named "outline". If either is unavailable, stop immediately and report which one.

Constants:
- INBOX: Drive folder "00_受信箱", ID 11_RuOPedK3r8koOu519nULsmTR5WDCFu (in My Drive root).
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
   b. Read the PDF. Use read_file_content first. If the text is missing or clearly incomplete (scanned images), download the file and read the pages visually. Never guess content you cannot read.
   c. Extract: 書類日付 (the document's issue date; if unreadable, use the file's createdTime converted to JST and mark it "（スキャン日）"), 書類名 (short descriptive title), 種別 (通知/請求書/領収書/契約書/証明書/明細/申請書/その他), 発行元, 期限・金額 (deadlines, amounts, validity periods; "なし" if none), 要約 (2–4 sentences: what it is, from whom, required action and deadline), and a full Markdown transcription that keeps headings, lists and tables, with pages separated by "### p.N".
   d. Mask sensitive numbers everywhere you write them, keeping only the last 4 digits as "****1234": My Number (マイナンバー), full credit card numbers, bank account numbers, passwords and PINs, 基礎年金番号, health insurance 記号・番号, driver's licence numbers, passport numbers.
   e. Decide the destination: the best-fitting existing subfolder of a CATEGORY FOLDER. If none fits, you may create one new subfolder under the best-fitting CATEGORY FOLDER, named like the existing ones (a short Japanese noun such as 税務, 保険). A document that fits a category but no specific topic may go directly in the category folder. If you cannot decide with confidence (unreadable scan, several unrelated documents in one PDF), the destination is "要振り分け": the file stays in INBOX with its original name.
      Title = "YYYYMMDD_書類名" using 書類日付 (replace "/" and other characters invalid in names with "_"). File name = title + ".pdf".
   f. Create the Outline document FIRST, in collection "書類", nested under parent documents that mirror the Drive path (e.g. 書類 / 06_諸手続き / 税務). A parent document's title is exactly the folder name and its body is a link to that Drive folder; create missing parents. Put "要振り分け" documents under a parent titled "00_受信箱（要振り分け）". If the Outline tools cannot nest documents, create it directly in the collection and say so in the report.
      Document title: the title from step e. Body, exactly in this shape:
        > このページは自動取り込みで作成されました。機微な番号は伏せ字にしています。原本は下の Drive リンクを参照してください。
        a two-column table "| 項目 | 値 |" with rows: 書類日付, 書類名, 種別, 発行元, 期限・金額, 原本 (a Markdown link whose text is the new file name and whose URL is https://drive.google.com/file/d/<FILE_ID>/view), 保管場所 ("Drive: <category>/<subfolder>" or "要振り分け"), Drive ファイル ID (the ID in backticks), スキャン日時 (createdTime in JST), 取り込み日時 (now, JST)
        "## 要約" followed by the summary
        "## 本文（書き起こし）" followed by the transcription
   g. Only after the document exists: if the destination is not "要振り分け", rename the Drive file to the new file name and move it into the destination folder (one update that sets title and parentId). Then confirm the file's parent is the destination.
   h. If any step fails for a file, record the error and continue with the next file. A file whose document was not created stays in INBOX and is retried in full next run.
4. Safety rules: never delete or trash anything in Drive or Outline. Never touch Drive files other than the one being filed. Never edit Outline documents except ones this job created (they contain "Drive ファイル ID") and the folder parent documents.
5. Final report (your last message), in Japanese: a table with one row per processed file — 元のファイル名, 新しいファイル名, 振り分け先, Outline のリンク, 結果 (登録・移動済み / 移動のみ再実行 / 要振り分け / エラー: reason). Then list: new Drive folders created, files skipped because of MAX_FILES, files still needing manual filing, and non-PDF files in INBOX. If INBOX had nothing to do, say so in one line.
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

### Re-ingest a file

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
