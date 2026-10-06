# Tender Document Package Builder

A frontend-only web application that helps an office worker assemble a tender submission. The user loads a `requirements.json` file, uploads PDF documents, matches each PDF to a required document, enters expiry dates, fixes validation problems, and downloads one combined PDF package with a cover page, correct ordering, and page-numbered footers.

Built for the AI DevFest 2026 Vibe Coding Competition.

## Privacy: browser-only processing

- The application is **frontend-only**. There is no backend, database, login, or cloud storage in this project.
- All PDF reading, hashing, validation, and generation happens **inside the browser** on the user's machine.
- The code contains no logic that uploads tender documents anywhere.
- The only external resource the page loads is the `pdf-lib` script from `cdnjs.cloudflare.com` (see [Technology](#technology)). Documents are never sent to it.

## Technology

| Item | Detail |
|---|---|
| Application | One self-contained file: `tender-package-builder.html` (HTML, CSS, vanilla JavaScript) |
| PDF library | [pdf-lib](https://pdf-lib.js.org/) 1.17.1, loaded from `https://cdnjs.cloudflare.com/ajax/libs/pdf-lib/1.17.1/pdf-lib.min.js` |
| Hashing | Browser Web Crypto API (`crypto.subtle.digest`, SHA-256) |
| Framework / build step | None |
| Target browser | Latest Google Chrome |

pdf.js is **not** used. Page counting and package generation are both done with pdf-lib.

## Running it

No installation or build is required.

1. Open `tender-package-builder.html` in Google Chrome. An internet connection is needed the first time so the pdf-lib script can load from the CDN.
2. Follow the workflow below.

If the page is hosted somewhere that provides the Claude Artifact `downloads` API, the final download goes through that API's save prompt. Everywhere else it falls back to a standard browser download of `<tender_id>_Package.pdf`.

## Workflow

1. **Load requirements:** choose `requirements.json`.
2. **Upload PDFs:** select multiple PDFs at once (or drag them onto the upload box).
3. **Match documents:** for each requirement, pick the uploaded file from its dropdown.
4. **Enter expiry dates:** shown only for requirements with `has_expiry: true` that have a file.
5. **Fix problems:** the status badges and the "why generation is disabled" panel show what is blocking.
6. **Generate:** the button enables once nothing is blocking.
7. **Download:** the package is saved as `<tender_id>_Package.pdf`. A "Download again" button remains until the user changes any match, date, or file.

## `requirements.json` format

```json
{
  "tender": {
    "tender_id": "...",
    "title": "...",
    "procuring_entity": "...",
    "bidder": "...",
    "submission_deadline": "YYYY-MM-DD"
  },
  "requirements": [
    {
      "id": "...",
      "order": 1,
      "title_en": "...",
      "title_bn": "...",
      "mandatory": true,
      "has_expiry": true
    }
  ]
}
```

Everything is driven by this file and by the uploaded PDFs. No document names, IDs, titles, or dates are hard-coded, and any number of requirements is supported.

Loading rules, as implemented:

- `tender_id` is required and must be non-empty.
- `submission_deadline` must be a real calendar date in `YYYY-MM-DD` form. A trailing time part is accepted and ignored.
- Each requirement must have an `id`. IDs are treated as strings, may contain any characters, and must be unique.
- Requirements are sorted by `order`. If `order` is missing or not numeric, the requirement's position in the file is used.
- `mandatory` and `has_expiry` are true for `true`, `"true"`, `1`, or `"1"`, and false otherwise.
- If `title_bn` is missing, `title_en` is used (and the reverse for `title_en`).
- An invalid file shows a clear error in the current UI language.

## Features implemented

### PDF upload

- Multiple files at once; non-PDF files are rejected with a message.
- Limits: **30 files** and **50 MB total** (52,428,800 bytes). Files over a limit are not added, and the reason is shown per file.
- Each file shows name, size, page count, matched requirement, duplicate state, and a remove button. A running "files / total size" summary is shown.
- Files are validated and counted one at a time and the list updates as each file is read. Overlapping selections are queued so limits stay exact.
- Damaged, truncated, empty, and password-protected PDFs are rejected with a specific message and do not crash the app. PDFs that are encrypted without needing a password to open (owner-password only) are also rejected as password protected.

### Exact duplicate detection

- Every file is hashed with SHA-256. Files with identical content are flagged with a "Duplicate group" badge that names the identical files.
- An identical copy cannot be matched to a different requirement than its twin. Duplicates are shown, never silently ignored.
- Duplicates are a warning, not a blocking status.

### Matching

- One requirement has at most one file, and one file has at most one requirement.
- A match can be changed or undone. Picking a file already used by another requirement moves it and frees the old requirement.
- Undoing a match, replacing the file, or removing the file also clears that requirement's expiry date, so a stale date cannot carry over to a different document.

### Validation

Each requirement has exactly one status, recalculated after every change:

| Status | Rule | Blocks generation |
|---|---|---|
| Missing | `mandatory` and no file matched | Yes |
| Expiry date needed | `has_expiry`, file matched, no expiry entered | Yes |
| Expired | `has_expiry`, file matched, expiry before `submission_deadline` | Yes |
| Not provided | not `mandatory` and no file matched | No |
| OK | file matched and, if an expiry is required, expiry on or after `submission_deadline` | No |

An expiry equal to `submission_deadline` is **OK**. Dates are compared as plain `YYYY-MM-DD` values, so time zones do not affect the result.

The **Generate** button is disabled while any blocking status exists, and the page lists the reasons with counts.

### Package generation

The output file is `<tender_id>_Package.pdf` (characters other than letters, digits, `_`, `.`, and `-` in the ID are replaced with `_` in the filename).

1. **Page 1, English cover page:** tender ID, title, procuring entity, bidder, submission deadline, package creation date (the user's local date), and the included documents in required order with their page ranges in the package.
2. **Matched documents only**, sorted by `order`, with every page of every source PDF in its original order. Optional requirements without a file are skipped.
3. **Footer on every page, including the cover:** `<tender_id> | Page X of Y`, where Y is the total number of pages in the final package.

How footers avoid covering content: for source pages, a 34-point strip is added outside the page's visible area (below it, adjusted for page rotation) and the footer is drawn there. Source content is not scaled or moved, so output pages are 34 points larger than their source pages. On the cover, the footer sits in the bottom margin.

After building, the app re-opens the generated PDF and checks that its page count matches the expected total.

### Bilingual UI

- English and Bangla, switched with a prominent toggle in the header.
- All interface text, statuses, and error messages follow the selected language. Document names use `title_en` or `title_bn`.
- The generated cover and footer are always English.

## Code organisation

The application is a single file, but the script is divided into clearly separated sections. Business logic does not depend on the UI code.

| Section | Responsibility |
|---|---|
| `I18N` | English and Bangla text |
| `Req`, `Day`, `Err` | Parsing and validating `requirements.json`; date normalisation; translatable errors |
| `Pdf` | SHA-256 hashing, PDF validity and page counting |
| `S`, `Model` | State (files, matches, expiry dates, language) and matching rules |
| `Intake` | Upload pipeline: type check, limits, validation, page count, hash |
| `Val` | Status engine and blocking counts |
| `Gen` | Cover page, page assembly, footers, output check |
| `UI` | Rendering |
| `App` | Event handling, upload, generate, download |

## Limitations and not implemented

Not implemented:

- Bonus features: index page, PNG seal or signature placement, CSV/Excel checklist export, save and reopen, Bangla text inside the PDF, filename-based auto-match suggestions, and AI assistance.
- Persistence: state is held in memory and is lost when the page is reloaded.
- Automated tests are not included in this repository.

Known limitations:

- Non-Latin characters (for example Bangla in an English title field) print as `?` on the cover and in footers, because the built-in PDF font cannot draw them.
- Form fields and bookmarks in source PDFs are not carried into the package. Page content, links, and annotations are copied.
- A PDF that is structurally valid but has corrupted page content cannot be detected during upload and is copied as it is.
- Very large inputs near the 50 MB limit are processed in browser memory and may be slow on low-memory machines.

## Testing status

During development the logic was exercised with scripted tests (matching, validation edge cases including expiry equal to the deadline, upload limits, duplicate detection, damaged and encrypted PDFs, and package generation with rotated, cropped, and many-page inputs), and with a headless DOM run of the page. Those scripts are not part of this repository, and the application has not been verified in this README's author's environment beyond that. Please run a manual check in Chrome before relying on it:

1. Load a `requirements.json` and upload all PDFs; check names, sizes, and page counts.
2. Upload a renamed copy of a file and confirm the duplicate badge and the matching restriction.
3. Upload a non-PDF, a damaged PDF, and a password-protected PDF.
4. Match documents; enter expiry dates before, equal to, and after the deadline.
5. Confirm Generate is disabled while anything is Missing, Expiry date needed, or Expired.
6. Generate and open the PDF: check the cover, document order, total page count, and the footer on every page.
7. Switch to Bangla and check the interface and error messages.
