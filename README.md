# Tender Document Package Builder

A frontend-only web application that helps an office worker assemble a tender submission. The user loads a `requirements.json` file, uploads PDF documents, matches each PDF to a required document, verifies or edits automatically detected expiry dates, fixes validation problems, and downloads one combined PDF package with an English cover page, correct ordering, and page-numbered footers.

Built for the AI DevFest 2026 Vibe Coding Competition.

---

## Privacy: 100% Browser-Only Processing

* The application is **strictly frontend-only**. There is no backend, database, authentication, or cloud storage.
* All PDF parsing, hashing, text extraction, expiry date detection, validation, and package generation happen **locally inside the browser** on the user's machine.
* The code contains no logic that transmits tender documents, text, or metadata anywhere.
* The only external resources loaded are CDN scripts: `pdf-lib` and `pdf.js` (see [Technology](#technology)). Documents and extracted text are never sent to external servers or AI APIs.

---

## Technology

| Item | Detail |
|---|---|
| **Application** | Self-contained single-page application: `index.html` (HTML5, Vanilla CSS, Vanilla JavaScript) |
| **PDF Assembly & Generation** | [pdf-lib](https://pdf-lib.js.org/) 1.17.1 from `https://cdnjs.cloudflare.com/ajax/libs/pdf-lib/1.17.1/pdf-lib.min.js` |
| **PDF Text Extraction** | [pdf.js](https://mozilla.github.io/pdf.js/) 3.11.174 from `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js` |
| **File Hashing** | Browser Web Crypto API (`crypto.subtle.digest`, SHA-256) |
| **Styling** | Vanilla CSS design tokens with full light/dark mode support |
| **Framework / Build Step** | None (zero dependencies, zero build step) |
| **Target Browser** | Modern Google Chrome |

---

## Running the Application

No installation or build step is required.

1. Open `index.html` in Google Chrome (an active internet connection is required on first load to fetch `pdf-lib` and `pdf.js` from cdnjs).
2. Follow the workflow below.

If the page is hosted in an environment providing the Claude Artifact `downloads` API, the final download uses that prompt; in standard browsers, it triggers a native file download of `<tender_id>_Package.pdf`.

---

## Workflow

1. **Load requirements:** Click **Open requirements.json** (or **requirements.json খুলুন**).
2. **Upload PDFs:** Select multiple PDFs at once (or drag-and-drop onto the drop zone).
3. **Match documents:** For each requirement, select the uploaded file from its dropdown.
4. **Automatic Expiry Date Detection & Confirmation:**
   * For requirements with `has_expiry: true`, matching a PDF immediately triggers in-browser text extraction and date analysis.
   * If a high-confidence expiry date is detected, the date input is automatically populated, displaying a green detection banner with **[Use this date]** and **[Edit]** options.
   * If multiple or lower-confidence dates are found, interactive buttons are shown under **Possible expiry dates:** for the user to select.
   * If the PDF is scanned or lacks extractable dates, a clear manual entry prompt is shown.
   * The user can manually edit or change the date at any time.
5. **Fix problems:** Check badges and the blocking issues alert box (Missing, Expiry date needed, Expired).
6. **Generate:** The **Generate & download package** button activates once all blocking issues are resolved.
7. **Download:** The combined package is saved as `<tender_id>_Package.pdf`. A **Download again** button remains active until matches, files, or dates change.

---

## `requirements.json` Format

```json
{
  "tender": {
    "tender_id": "TND-2026-001",
    "title": "School Infrastructure Construction",
    "procuring_entity": "Ministry of Primary and Mass Education",
    "bidder": "Acme Builders Ltd",
    "submission_deadline": "2026-10-15"
  },
  "requirements": [
    {
      "id": "trade_license",
      "order": 1,
      "title_en": "Trade License",
      "title_bn": "ট্রেড লাইসেন্স",
      "mandatory": true,
      "has_expiry": true
    },
    {
      "id": "tax_cert",
      "order": 2,
      "title_en": "TIN / Tax Clearance Certificate",
      "title_bn": "কর পরিশোধের সনদ",
      "mandatory": true,
      "has_expiry": true
    },
    {
      "id": "bank_solvency",
      "order": 3,
      "title_en": "Bank Solvency Certificate",
      "title_bn": "ব্যাংক সচ্ছলতা সনদ",
      "mandatory": false,
      "has_expiry": false
    }
  ]
}
```

Loading rules:
* `tender.tender_id` is mandatory and cannot be blank.
* `tender.submission_deadline` must be a valid calendar date in `YYYY-MM-DD` format (trailing times are safely ignored).
* Each requirement must have a unique `id`.
* Requirements are ordered by `order` (falls back to file order if missing).
* `mandatory` and `has_expiry` accept boolean `true`/`false`, strings `"true"`/`"false"`, or `1`/`0`.
* If `title_bn` is omitted, `title_en` is used (and vice-versa).

---

## Features Implemented

### 1. Automatic Expiry Date Detection (New)

* **Client-Side Text Extraction**: Uses `pdf.js` to extract text from all pages of matched PDFs on demand.
* **Smart In-Memory Caching**: Extracted text is cached on the file object (`file._text`). Changing requirements or re-rendering never re-parses the PDF unnecessarily.
* **Numeral Normalization**: Recognizes Bangla numerals (`০, ১, ২, ৩, ৪, ৫, ৬, ৭, ৮, ৯`) and normalizes them 1-to-1 to standard digits (`0-9`), preserving offsets and Bangla text.
* **Bangla Month Names & Spelling Variations**:
  * Supports full names and common variants: জানুয়ারি/জানুয়ারি, ফেব্রুয়ারি/ফেব্রুয়ারি, মার্চ, এপ্রিল, মে, জুন, জুলাই, আগস্ট/অগাস্ট, সেপ্টেম্বর, অক্টোবর, নভেম্বর, ডিসেম্বর.
* **Date Formats Supported**:
  * `DD/MM/YYYY`, `DD-MM-YYYY`, `DD.MM.YYYY`
  * `YYYY-MM-DD`, `YYYY/MM/DD`, `YYYY.MM.DD`
  * `DD Month YYYY` (e.g. `20 October 2027`, `20th Oct 2027`, `২০ অক্টোবর ২০২৭`, `২০শে অক্টোবর ২০২৭`)
  * `Month DD, YYYY` (e.g. `October 20, 2027`, `Oct 20, 2027`, `অক্টোবর ২০, ২০২৭`)
* **Context & Proximity Scoring**:
  * **High Confidence**: Triggered when strong expiry phrases immediately precede or neighbor a valid date (e.g., `Expiry Date`, `Valid Until`, `Valid Till`, `Valid Up To`, `Expires`, `মেয়াদ শেষের তারিখ`, `মেয়াদ উত্তীর্ণের তারিখ`, `মেয়াদোত্তীর্ণের তারিখ`).
  * **Medium Confidence**: Triggered by validity phrases (e.g., `Validity`, `বৈধতার তারিখ`, `বৈধতার মেয়াদ`, `মেয়াদ`).
  * **Issue Date Disambiguation**: Actively penalizes issue date phrases (e.g., `Date of Issue`, `Issue Date`, `ইস্যুর তারিখ`, `প্রদানের তারিখ`) to ensure issue dates are never selected as expiry dates.
* **Safe, Transparent UI**:
  * **High confidence**: Auto-populates the date field, displays `✓ Expiry date detected automatically` / `✓ মেয়াদ শেষের তারিখ স্বয়ংক্রিয়ভাবে সনাক্ত হয়েছে`, the formatted date, and **[Use this date]** + **[Edit]** buttons.
  * **Lower confidence / Multiple dates**: Displays candidate buttons under `Possible expiry dates:` / `সম্ভাব্য মেয়াদ শেষের তারিখ:` allowing single-click selection.
  * **No date found**: Displays informative note: `Expiry date could not be detected automatically. Please enter it manually.` (non-blocking).
  * **Scanned / Image-only PDF**: Gracefully detects when no text layer exists and shows: `Automatic expiry detection is unavailable for this scanned PDF. Please enter the expiry date manually.`
  * **Manual Edits**: The user can override or fine-tune any date using the native HTML date picker.

### 2. PDF Upload & Validation

* Multi-file upload via file picker or drag-and-drop.
* Limits enforced: **up to 30 files** and **50 MB total** (52,428,800 bytes).
* Upload queue processes files sequentially to ensure limits and page counts remain accurate.
* Rejects non-PDFs, empty files, corrupted documents, and password-protected files with specific error messages.

### 3. Exact Duplicate Detection

* Calculates SHA-256 hashes via `crypto.subtle.digest`.
* Files with identical binary content are grouped under a lettered badge (e.g., `Duplicate group A`).
* Identical copies are restricted from being matched to different requirements simultaneously.

### 4. Document Matching & Integrity

* 1-to-1 matching: one requirement has at most one file, and one file has at most one requirement.
* Moving or undoing a match automatically cleans up associated expiry dates and detection state, preventing stale dates from bleeding into other documents.

### 5. Status & Validation Engine

Each requirement is evaluated continuously:

| Status | Rule | Blocks Package Generation? |
|---|---|---|
| **Missing** | `mandatory: true` and no file matched | Yes |
| **Expiry date needed** | `has_expiry: true`, file matched, no date entered/chosen | Yes |
| **Expired** | `has_expiry: true`, file matched, expiry date strictly before `submission_deadline` | Yes |
| **Not provided** | `mandatory: false` and no file matched | No |
| **OK** | File matched and, if expiry is required, date is on or after `submission_deadline` | No |

* **Expiry equal to submission deadline is OK**.
* Comparisons use plain `YYYY-MM-DD` strings, ensuring time zone neutrality.

### 6. Package Generation

* Creates `<tender_id>_Package.pdf` using `pdf-lib`.
* **Cover Page (Page 1)**: Tender ID, Title, Procuring Entity, Bidder, Submission Deadline, Generation Date, and table of included documents with their page ranges.
* **Order Preservation**: Documents appear in required `order`, preserving all original source pages.
* **Non-Overlapping Footers**: On source pages, a 34-point extension strip is added below the page (respecting rotation) where `<tender_id> | Page X of Y` is drawn. The original content is never obscured.
* Post-generation verification validates that the final document's page count matches the expected total.

### 7. Bilingual UI

* Instant switching between **English** and **বাংলা** via the header toggle.
* All labels, statuses, instructions, detection banners, and error messages update immediately.
* Cover page and footers in the generated PDF remain in English for official submission standards.

---

## Code Organisation

| Section / Object | Responsibility |
|---|---|
| `I18N` | English and Bangla localization strings for all UI states and detection notices |
| `BN_DIGITS` / `toBnDigits` / `bnToEnDigits` | Numeral bidirectional converters for Bangla and Western digits |
| `DateParser` | Candidate extraction (regexes), Bangla month maps, calendar validation, proximity scoring |
| `Extractor` | Asynchronous browser-side PDF text extraction using `pdf.js` with per-file caching |
| `Req`, `Day`, `Err` | Parsing and validation of `requirements.json`; date normalizer |
| `Pdf` | SHA-256 hashing, PDF validation, and page count inspection |
| `S`, `Model` | Application state (files, matches, expiries, detections) and matching rules |
| `Intake` | Upload queue, file validation, limit enforcement, and hashing pipeline |
| `Val` | Core status engine and blocking issue counts |
| `Gen` | Package builder: cover page, page copying, rotation handling, margin strip footers |
| `UI` | DOM rendering for header, file list, requirements table, detection banners, and gates |
| `App` | Controller: event handling, upload, match, expiry detection, date picking/confirmation, generation |

---

## Limitations

* **OCR / Scanned Documents**: Image-only scans without an embedded text layer cannot be parsed for text; the app prompts the user to enter the date manually.
* **Cover & Footer Glyphs**: Built-in Helvetica fonts in the PDF generator only draw Latin characters; non-Latin characters in titles fallback to `?` on the generated cover/footer.
* **Forms & Bookmarks**: Form field values and bookmarks from source PDFs are flattened or omitted in the combined output; page content and annotations are preserved.
* **Browser Memory**: State is held in browser memory and resets on page reload.

---

## Verification & Manual Testing Guide

To verify the application in Google Chrome:

1. **English Expiry Phrase + English Date**:
   * Load a `requirements.json` with `submission_deadline: "2026-10-15"`.
   * Match a PDF containing: `"Valid Until: 20/10/2027"`.
   * Verify date auto-populates as `2027-10-20` with `✓ Expiry date detected automatically` and status **OK**.
2. **Bangla Expiry Phrase + Bangla Numerals**:
   * Match a PDF containing: `"মেয়াদ শেষের তারিখ: ২০/১০/২০২৭"`.
   * Verify it normalizes and populates `2027-10-20`. Switch UI language to **বাংলা** and verify localized text.
3. **Issue Date vs Expiry Date Disambiguation**:
   * Match a PDF containing: `"Date of Issue: 20/10/2025\nValid Until: 20/10/2027"`.
   * Verify the system selects `20/10/2027` (HIGH confidence) and places `20/10/2025` under "Other dates".
4. **Bangla Issue Date + Bangla Expiry Date**:
   * Match a PDF containing: `"ইস্যুর তারিখ: ২০/১০/২০২৫\nমেয়াদ শেষের তারিখ: ২০/১০/২০২৭"`.
   * Verify `20/10/2027` is selected.
5. **Multiple Candidate Dates / Suggestions**:
   * Match a PDF with multiple dates without strong keywords (e.g. `"Audit dates: 20/10/2027 and 20/10/2028"`).
   * Verify the field remains blank, displaying candidate buttons under `Possible expiry dates:`. Click a button to populate.
6. **No Expiry Date**:
   * Match a PDF with no dates.
   * Verify message: `"Expiry date could not be detected automatically. Please enter it manually."`
7. **Scanned / Image-Only PDF**:
   * Match a scanned image-only PDF with no extractable text.
   * Verify notice: `"Automatic expiry detection is unavailable for this scanned PDF. Please enter the expiry date manually."`
8. **Expiry Equal to Submission Deadline**:
   * Enter expiry equal to deadline (`2026-10-15`). Verify status badge shows **OK**.
9. **Expired Document**:
   * Enter expiry before deadline (`2026-05-10`). Verify status badge shows **Expired** and generation is blocked.
10. **Future Expiry Date**:
    * Enter expiry after deadline (`2027-10-20`). Verify status badge shows **OK**.
11. **User Edits Detected Date**:
    * Click **[Edit]** or pick a date in the date picker. Verify the status updates and remains confirmed.
12. **Various Date Formats**:
    * Test `20-10-2027`, `20.10.2027`, `2027-10-20`, `20 October 2027`, and `October 20, 2027`.
    * Verify all correctly normalize to `2027-10-20`.
