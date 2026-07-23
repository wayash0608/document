# REX Dynamic Checklist Form Builder — Technical Specification

**Status:** Draft for review
**Author:** Retail Doctor Group
**Last updated:** 2026-07-23

---

## 1. Purpose

Today, onboarding a new client checklist is a manual developer task: build the Gravity Form by hand, write a client PHP class, write a report template, and wire it into `vantage-it.php`. As RDG onboards more clients, this does not scale.

This document specifies **Phase 1: the Excel → Gravity Form generator** — an admin uploads a spreadsheet plus a few settings, and the system creates the Gravity Form automatically via `GFAPI::add_form()`.

> **Scope note:** This spec covers *creating the form*. The *processing* side (scoring, PDF, email, report page) is a separate but related phase. The scoring is designed to work from the radio choice values defined here, so no per-client scoring code is needed.

---

## 2. Excel file format

The admin-supplied spreadsheet has these columns:

| No. | Category | Checklist Item | What to Check | Input Type | Image | Marks |
|-----|----------|----------------|---------------|-----------|-------|-------|
| 1 | Visual Merchandising | Storefront condition & branding consistency | The storefront, entrance and external signage are clean… | YN | Y | 3 |
| 2 | Visual Merchandising | Window displays and Promotional Displays | Window glass and displays are clean and free of dust… | YN | Y | *(empty)* |
| … | … | … | … | … | … | … |
| 6 | Customer Experience | Warranty and installation implications | The team member clearly explains applicable warranty… | YN | N | 5 |

### Column meaning

| Column | Meaning | Maps to |
|--------|---------|---------|
| **No.** | Display order (informational) | Field order within page |
| **Category** | Groups questions; each category becomes its own page | Page title / section |
| **Checklist Item** | The question | Field **label** |
| **What to Check** | Guidance text shown under the question | Field **description** |
| **Input Type** | `YN` = Yes/No radio | Gravity Forms `radio` field |
| **Image** | `Y` = attach a file upload after the question; `N` = none | Gravity Forms `fileupload` field |
| **Marks** | The score for a **Yes** answer. **Empty = unscored** (no marks assigned to that question). | Radio choice `value` encoding |

---

## 3. Admin UI — settings asked at the top (not from Excel)

Before/alongside the upload, the admin provides:

| Setting | Gravity Forms property | Example |
|---------|------------------------|---------|
| Form name | `form.title` | `Floorworld Store Audit Checklist` |
| Required indicator = asterisk | `form.requiredIndicator = 'asterisk'` | — |
| Require user to be logged in | `form.requireLogin = true` | — |
| Login message (HTML) | `form.requireLoginMessage` | see below |
| CSS Class Name | `form.cssClass` | `floorworld-store-audit-checklist` |

**Login message HTML** (stored verbatim in `requireLoginMessage`):

```html
<div class="alert bg-info-subtle border border-1 border-info m-5 p-5 rounded-3">
  Please <a href="https://demo.rdgrx.com.au/wp-login.php?itsec-hb-token=panel&redirect_to=https%3A%2F%2Fdemo.rdgrx.com.au%2Fdfloorworld-store-audit-checklist%2F">log in</a> to fill this checklist.
</div>
```

---

## 4. Fixed first page — "Store details"

This page is **always generated identically**, before any Excel content. It is NOT taken from the spreadsheet.

**Page 1 title:** `Store details`

| Field | GF Type | Settings |
|-------|---------|----------|
| Store | `select` | `cssClass = 'populate-locations'`, required. Populated at render time by the existing `gform_pre_render` store-dropdown hook. |
| Store Owner Name | `hidden` | Placeholder for now (may be auto-filled later from the selected store's location record). |
| Company Name | `hidden` | Placeholder for now. |

A **page break** follows this page.

---

## 5. Excel content → form structure

### 5.1 Paging rule
- Rows are **grouped by `Category`**, preserving Excel order.
- **Each Category becomes its own page.** A page break is inserted between categories.
- The page title = the Category name.

### 5.2 Per-row field mapping

For every spreadsheet row, generate a **radio field**, and (if `Image = Y`) a **file upload field** directly after it.

#### Radio field (Input Type = `YN`)

| Property | Value |
|----------|-------|
| `type` | `radio` |
| `label` | `Checklist Item` (column 3) |
| `description` | `What to Check` (column 4) |
| `cssClass` | `yes-no-toggle` |
| `isRequired` | `true` |
| `choices` | see encoding below |

**Choice encoding — driven by the `Marks` column (NOT hardcoded):**

The mark for a **Yes** answer comes from the spreadsheet's `Marks` column. It is never entered by hand in the builder.

**Case A — `Marks` has a value (e.g. `3`):**

| Choice text | `value` | Score |
|-------------|---------|-------|
| Yes | `b-{Marks}` (e.g. `b-3`) | value of `Marks` |
| No | `a-0` | 0 |

**Case B — `Marks` is empty → unscored question:**

| Choice text | `value` | Score |
|-------------|---------|-------|
| Yes | `Yes` | — (not scored) |
| No | `No` | — (not scored) |

> When a mark is present, the number after the dash is the **score**, and the `a-` / `b-` prefixes keep a stable sort order. A generic scoring engine reads the value, splits on `-`, and totals the numeric part — no per-client code required. When `Marks` is empty, plain `Yes` / `No` values are used and the question is excluded from scoring totals.

#### File upload field (only when Image = `Y`)

| Property | Value |
|----------|-------|
| `type` | `fileupload` |
| `label` | **(empty — label removed)** |
| `labelPlacement` / visibility | Label hidden |
| Association | Belongs to the question immediately above it |

When `Image = N`, no file upload field is created for that row.

---

## 6. Worked example — Floorworld

Given the sample spreadsheet, the generated form is:

```
FORM: "Floorworld Store Audit Checklist"
  requiredIndicator: asterisk
  requireLogin: true
  requireLoginMessage: <div class="alert …">Please log in…</div>
  cssClass: floorworld-store-audit-checklist

PAGE 1 — "Store details"
   • Store              (select, populate-locations, required)
   • Store Owner Name   (hidden)
   • Company Name       (hidden)
--- page break ---
PAGE 2 — "Visual Merchandising"
   1. Storefront condition & branding consistency   (radio yes-no-toggle, required) + File Upload
   2. Window displays and Promotional Displays      (radio yes-no-toggle, required) + File Upload
   3. Accessibility (parking, entry points, etc.)   (radio yes-no-toggle, required) + File Upload
   4. Renovator                                     (radio yes-no-toggle, required) + File Upload
--- page break ---
PAGE 3 — "Customer Experience"
   5. Technical attributes (wear layer, AC rating…) (radio yes-no-toggle, required) + File Upload
   6. Warranty and installation implications        (radio yes-no-toggle, required)   [no image]
   7. Booking an in-home quote when in-store         (radio yes-no-toggle, required)   [no image]
   8. Accuracy and clarity of quotes                 (radio yes-no-toggle, required) + File Upload
```

---

## 7. Gravity Forms field structure (JSON produced per element)

The builder assembles a `$form` array and calls `GFAPI::add_form($form)`. Illustrative shapes:

### Form envelope
```json
{
  "title": "Floorworld Store Audit Checklist",
  "requiredIndicator": "asterisk",
  "requireLogin": true,
  "requireLoginMessage": "<div class=\"alert bg-info-subtle …\">Please log in…</div>",
  "cssClass": "floorworld-store-audit-checklist",
  "fields": [ /* see below */ ],
  "pagination": { "type": "percentage" }
}
```

### Store dropdown (page 1)
```json
{
  "type": "select",
  "label": "Store",
  "cssClass": "populate-locations",
  "isRequired": true,
  "choices": []
}
```

### Hidden fields (page 1)
```json
{ "type": "hidden", "label": "Store Owner Name" }
{ "type": "hidden", "label": "Company Name" }
```

### Page break
```json
{ "type": "page" }
```
> The page **title** (e.g. "Store details", "Visual Merchandising") is set via the page field's `nextButton`/`cssClass` conventions or a Section field at the top of each page, depending on the chosen GF rendering approach.

### Radio question — with a mark (`Marks = 3`)
```json
{
  "type": "radio",
  "label": "Storefront condition & branding consistency",
  "description": "The storefront, entrance and external signage are clean…",
  "cssClass": "yes-no-toggle",
  "isRequired": true,
  "choices": [
    { "text": "Yes", "value": "b-3" },
    { "text": "No",  "value": "a-0" }
  ]
}
```

### Radio question — no mark (`Marks` empty, unscored)
```json
{
  "type": "radio",
  "label": "Window displays and Promotional Displays",
  "description": "Window glass and displays are clean and free of dust…",
  "cssClass": "yes-no-toggle",
  "isRequired": true,
  "choices": [
    { "text": "Yes", "value": "Yes" },
    { "text": "No",  "value": "No"  }
  ]
}
```

### File upload (only when Image = Y, label removed)
```json
{
  "type": "fileupload",
  "label": "",
  "labelPlacement": "hidden_label",
  "isRequired": false
}
```

> **Note:** Gravity Forms assigns numeric field IDs on save. After `GFAPI::add_form()` returns, re-read the form to capture the ID → question map, and store it (e.g. against the datasource record) for the processing phase.

---

## 8. Build algorithm (pseudocode)

```
read settings (name, login flag, login message, css class)
read spreadsheet rows

fields = []

# --- Fixed page 1 ---
fields += SectionTitle("Store details")
fields += Select("Store", cssClass="populate-locations", required=true)
fields += Hidden("Store Owner Name")
fields += Hidden("Company Name")
fields += PageBreak()

# --- Excel content ---
currentCategory = null
foreach row in rows:
    if row.Category != currentCategory:
        if currentCategory != null:
            fields += PageBreak()
        fields += SectionTitle(row.Category)
        currentCategory = row.Category

    if row.InputType == "YN":
        if row.Marks is not empty:
            choices = [ {Yes, "b-" + row.Marks}, {No, "a-0"} ]   # scored
        else:
            choices = [ {Yes, "Yes"}, {No, "No"} ]               # unscored

        fields += Radio(
            label       = row.ChecklistItem,
            description  = row.WhatToCheck,
            cssClass     = "yes-no-toggle",
            required     = true,
            choices      = choices
        )

    if row.Image == "Y":
        fields += FileUpload(label="")   # label removed

form = FormEnvelope(settings, fields)
form_id = GFAPI::add_form(form)
reload form  ->  capture field IDs  ->  save question map
```

---

## 9. Confirmed decisions

| # | Decision | Value |
|---|----------|-------|
| 1 | Yes score value | `b-{Marks}` — mark read from the `Marks` column (e.g. `b-3`) |
| 2 | No score value | `a-0` |
| 2a | `Marks` column empty | Unscored question — choices use plain `Yes` / `No` values, excluded from totals |
| 3 | Radio CSS class | `yes-no-toggle` |
| 4 | Radio required | Yes (asterisk) |
| 5 | Image = Y | File upload field, **label removed** |
| 6 | Image = N | No file upload field |
| 7 | First page | Fixed "Store details" (Store dropdown + 2 hidden fields) |
| 8 | Paging | One page per Category, page break between categories |

---

## 10. Open items / next phase

- **Store Owner Name / Company Name** — decide whether these are auto-populated from the selected store's `rdg-location` record, or remain blank placeholders.
- **File upload constraints** — max size / allowed types per existing checklist limits (`RDG_CHECKLISTS_MAX_IMAGE_SIZE_MB`, etc.).
- **Processing phase** — a generic engine that reads the `b-3` / `a-0` values to score submissions, generate the PDF/email, and drive the report page (removes the need for a per-client PHP class).
- **Datasource auto-creation** — after form creation, auto-create/link the `rdg-data-sources` record (form ID, report page, SMTP key, transient prefix) so the checklist is wired up with no code.
