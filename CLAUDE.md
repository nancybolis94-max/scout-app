# CLAUDE.md — قطاع الملايكة Scout App

> **Context file for Claude.** Read this entire file before making any change.

---

## Project Overview

Arabic RTL PWA for managing a Coptic Orthodox scout troop **"قطاع الملايكة"** (Angels Sector).
Two classes: **ملايكة أ** and **ملايكة ب**.

| Item | Value |
|---|---|
| Stack | Vanilla HTML + CSS + JS (single file) |
| Hosting | GitHub Pages |
| Database | Google Sheets via Apps Script proxy |
| File storage | Google Drive (PDFs) |
| PWA | No service worker yet |

---

## URLs

| Resource | URL |
|---|---|
| Live App | https://nancybolis94-max.github.io/scout-app/ |
| GitHub Repo | https://github.com/nancybolis94-max/scout-app |
| Google Sheet | https://docs.google.com/spreadsheets/d/1F_Dm2LUvXy0WPv3ygckwIY3_wgtj9-wlkqTOMz50USU/ |
| Drive PDF Folder | https://drive.google.com/drive/folders/10Q668J0Ckw2fo8J97s7rOF2KuCwpewOj |
| Apps Script (v3) | https://script.google.com/macros/s/AKfycbzjGKLbjSnmsuj3x5F-kDztIftSW7Cyhy3XtSEkWRrq8xeHPgvS0Qn1vkQKt_eWRSnyPw/exec |

---

## File Structure

```
scout-app/
├── index.html          ← entire app (~1800 lines)
├── CLAUDE.md           ← this file
└── apps_script_v3.gs   ← Apps Script source (reference copy)
```

Everything lives in `index.html`: HTML structure, CSS variables, and all JS.

---

## Google Sheet Structure (after mergeSheets())

### Tab: `القادة`
| A | B | C | D |
|---|---|---|---|
| الاسم | الفصل | الموبايل | تاريخ الميلاد |

Fصل values: `ملايكة أ` / `ملايكة ب` / `كلا الفصلين`

---

### Tab: `ملايكة أ` and `ملايكه ب`
Row 1 = headers. Data starts row 2.
| A | B | C | D | E | F |
|---|---|---|---|---|---|
| الاسم | العنوان | تاريخ الميلاد | تليفون الام | تليفون الاب | القائد المسئول |

Column F (القائد المسئول) is added by `mergeSheets()` — merges from old `توزيع الأولاد` tab.

---

### Tab: `خطة الاجتماعات` (19 columns after merge)
Row 1 = headers. Data starts row 2.
| Col | Field |
|---|---|
| A | رقم الاجتماع (e.g. "اجتماع 1") |
| B | التاريخ |
| C | مسئول أ (leader name) |
| D | مسئول ب (leader name) |
| E | روحية-موضوع |
| F | روحية-قائد أ |
| G | روحية-قائد ب |
| H | كشفية-موضوع |
| I | كشفية-قائد أ |
| J | كشفية-قائد ب |
| K | ثقافية-موضوع |
| L | ثقافية-قائد أ |
| M | ثقافية-قائد ب |
| N | لعبة-موضوع |
| O | لعبة-قائد أ |
| P | لعبة-قائد ب |
| Q | أخرى-موضوع |
| R | أخرى-قائد أ |
| S | أخرى-قائد ب |

**Before `mergeSheets()`** (old format, still supported by `loadAll`):
- A=رقم، B=تاريخ، C=مسئول، D=مسئول ب، E=روحية topic، F=كشفية topic، H=لعبة، I=كشفي
- No per-team leader columns

---

### Tab: `الأنشطة`
| A | B | C | D | E | F | G |
|---|---|---|---|---|---|---|
| الاسم | التاريخ | الفريق | قائد أ | قائد ب | PDF أ URL | PDF ب URL |

---

### Tab: `كلمات السر`
| A | B | C |
|---|---|---|
| الدور | الفصل | كلمة السر |

Roles: `قائد القطاع` / `قائد الفريق` / `مساعد قائد`

---

### Tabs removed after `mergeSheets()`
- `توزيع الخطة` → merged into `خطة الاجتماعات`
- `توزيع الأولاد` → merged into `ملايكة أ` / `ملايكه ب` col F

---

### Other tabs (read-only, append-only logs)
- `حضور القادة` — leader attendance log
- `افتقاد القادة` — leader visitation log
- `حضور الأولاد` — child attendance log
- `الافتقاد` — child followup log

---

## Apps Script (`apps_script_v3.gs`)

### `mergeSheets()` — run ONCE manually
Merges sheet structure (see above). Run from Apps Script Editor: **Run > mergeSheets**.

### `doGet(e)`
Query param: `?sheet=<sheetName>`
Returns: `{ data: [[row], [row], ...] }`

### `doPost(e)`
Body: JSON object. Supported actions:

```js
// Append new row
{ sheet: "القادة", action: "append", row: ["name", "cls", "phone", "dob"] }

// Update row (match by first column value)
{ sheet: "القادة", action: "update", match: "leader name", row: [...] }

// Delete row (match by first column value)
{ sheet: "القادة", action: "delete", match: "leader name" }

// Upload PDF to Drive, returns { status, fileId, viewUrl }
{ action: "uploadPDF", base64: "...", filename: "name.pdf" }
```

**CORS note:** All sheet writes use `mode: "no-cors"` → can't read response.
PDF upload uses `sheetPostWithResponse()` with `Content-Type: text/plain` → can read response.

---

## In-Memory Data Model (`DATA` object)

```js
DATA = {
  leaders: [
    { id: 1, name: "رانيا ماهر", cls: "ملايكة أ", phone: "...", dob: "..." }
  ],
  passwords: [
    { role: "قائد الفريق", cls: "ملايكة أ", password: "1234" }
  ],
  meetings: [
    {
      id: 1, num: 1, date: "2026-10-02",
      responsible_a: 3,   // leader id
      responsible_b: 11,  // leader id
      paras: {
        spiritual: { topic: "قصة آدم", leaders: [1,11], leaders_a: [1], leaders_b: [11] },
        scout:     { topic: "...",     leaders: [],      leaders_a: [],  leaders_b: [] },
        culture:   { topic: "...",     leaders: [],      leaders_a: [],  leaders_b: [] },
        game:      { topic: "...",     leaders: [],      leaders_a: [],  leaders_b: [] },
        other:     { topic: "...",     leaders: [],      leaders_a: [],  leaders_b: [] }
      }
    }
  ],
  activities: [
    {
      id: 1, name: "يوم كشفي", date: "2026-11-27", cls: "ملايكة أ",
      leader_a: 2, leader_b: 12,
      pdf_a: "https://drive.google.com/file/d/.../view",
      pdf_b: null
    }
  ],
  children: [
    { id: 1, name: "بارثينيا سعيد", cls: "ملايكة أ", lid: 3, phone: "...", addr: "...", dob: "..." }
    // ملايكة أ: ids 1–99
    // ملايكة ب: ids 1000+
  ],
  attendance: {},   // key: "att_{meetingId}_{childId}" → "present"/"excused"/"absent"
  followup:  {},    // key: "f_{childId}" → "done"/"pending"
  latt:      {},    // key: "la_{meetingId}_{leaderId}" → "present"/"excused"/"absent"
  lvisit:    {}     // key: "lv_{leaderId}" → "done"/"pending"
}
```

---

## Role System

### 3 Roles

| Role | Arabic | Login | Access |
|---|---|---|---|
| `admin` | قائد القطاع | Password only | Full access, all classes |
| `teamleader` | قائد الفريق | Class + password | Own class only |
| `assistant` | مساعد قائد | Class + name + password | Own assigned children only |

### `CU` (Current User) object
```js
CU = {
  role: "admin" | "teamleader" | "assistant",
  name: "string",
  cls: "ملايكة أ" | "ملايكة ب" | "",  // empty for admin
  id: 3  // leader id — only for assistant role
}
```

### Default passwords (from sheet `كلمات السر`)
- قائد الفريق ملايكة أ: `1234`
- قائد الفريق ملايكة ب: `1234`
- قائد القطاع: `1234` (was `Jesus2026`, changed in sheet)

---

## Tab Structure Per Role

### Admin tabs
`خطة الاجتماعات` | `الأولاد` | `القادة` | `حضور القادة` | `أعياد الميلاد` | `التقارير` | `كلمات السر`

### Team Leader tabs
`توزيع الخطة` | `الأولاد` | `حضور القادة` | `حضور الأولاد` | `الافتقاد` | `افتقاد القادة` | `الأنشطة` | `التقارير`

### Assistant tabs
`خطة السنة` | `الافتقاد` | `الأنشطة` | `التحضير`

---

## Key JS Functions Reference

### Data loading
| Function | Description |
|---|---|
| `loadAll()` | Fetches all sheet tabs in parallel, builds DATA. Called on page load. |
| `sheetGet(name)` | GET request to Apps Script for one sheet tab. Returns `[]` on failure. |
| `sheetPost(payload)` | POST no-cors write (no response). For all sheet writes. |
| `sheetPostWithResponse(payload)` | POST text/plain (readable response). For PDF upload only. |
| `uploadPdfToDrive(file, name, team)` | Reads file as base64, sends to Apps Script, returns Drive view URL. |

### Auth
| Function | Description |
|---|---|
| `selectRole(r)` | Switches login form between role tabs. |
| `doLogin()` | Validates password, sets `CU`, calls `initApp()`. |
| `doLogout()` | Clears `CU`, returns to login screen. |
| `getPass(role, cls)` | Looks up password from `DATA.passwords`, falls back to hardcoded. |

### Modal (IMPORTANT)
```js
// Always use this pattern — never set modal-ok.onclick directly
let _modalCb = null;
openModal("title", "html body", () => { /* save callback */ });
// modal-ok has ONE persistent addEventListener that calls _modalCb
```

### Rendering functions
| Function | Renders |
|---|---|
| `rAsPlan()` | Assistant: upcoming meeting + accordion |
| `rAsFollowup()` | Assistant: followup list |
| `rAsAttendance()` | Assistant: attendance |
| `rActivities(prefix)` | Activities for "as" prefix |
| `rTLPlan()` | TL: accordion plan with per-meeting save |
| `rTLChildren()` | TL: children table with leader assign |
| `rTLLatt()` | TL: leader attendance |
| `rTLCatt()` | TL: child attendance view |
| `rTLFollowup()` | TL: followup progress |
| `rTLLvisit()` | TL: leader visitation |
| `rTLActivities()` | TL: activities with PDF upload |
| `rAdPlan()` | Admin: meetings (upcoming + accordion) + activities |
| `rAdChildren()` | Admin: all children table |
| `rAdLeaders()` | Admin: leaders table with edit/delete |
| `rAdLatt()` | Admin: all leaders attendance |
| `rBdays()` | Admin: birthdays by month |
| `rReports(prefix)` | Reports for "ad" or "tl" prefix |
| `rPasswords()` | Admin: passwords management |

### Meeting helpers
| Function | Description |
|---|---|
| `parasHTML(m, highlight)` | Renders para cards for a meeting. `highlight=true` yellows current user's paras. |
| `adParaCards(m, inline)` | Admin version: `inline=true` for dark card (white text), `false` for normal. |
| `upcomingMeet()` | Returns the next upcoming meeting (or last if all past). |
| `toggleAcc(mid)` | Toggles assistant plan accordion. |
| `toggleAdAcc(mid)` | Toggles admin plan accordion. |
| `toggleTLAcc(mid)` | Toggles TL plan accordion. |

### Team helpers
| Function | Description |
|---|---|
| `teamKey(cls)` | `"ملايكة أ"` → `"leaders_a"`, else `"leaders_b"` |
| `getTeamLeaders(para, cls)` | Returns leader ids array for team from para object. |
| `setTeamLeaders(para, cls, ids)` | Sets leaders for team without touching other team. |
| `allParaLeaders(para)` | Union of leaders_a + leaders_b, deduped. |

---

## CSS Design Tokens

```css
--g:   #2D6A4F   /* primary green */
--gm:  #40916C   /* green medium */
--gl:  #74C69D   /* green light */
--gp:  #D8F3DC   /* green pale bg */
--gf:  #F0FAF3   /* green faint bg */
--am:  #E9A919   /* amber */
--ap:  #FFF3CD   /* amber pale */
--rd:  #C0392B   /* red */
--rp:  #FDECEA   /* red pale */
--bl:  #2563EB   /* blue (team A) */
--bp:  #EFF6FF   /* blue pale */
--tx:  #1A2E22   /* text primary */
--tm:  #3D5A47   /* text medium */
--mu:  #7A9585   /* text muted */
--bd:  #C8E6D0   /* border */
--bg:  #F4FAF6   /* page bg */
```

**Team colors:** فريق أ = `--bl` (blue) | فريق ب = `--g` (green)
**Mine highlight:** `background: #FFF176; border: 2px solid #F9A825`

---

## Important Patterns & Rules

### 1. Event listeners — NEVER inline onclick for edit/delete buttons
```js
// ✅ CORRECT
h += `<button class="ab ldr-edit-btn" data-lid="${l.id}">تعديل</button>`;
// after innerHTML:
c.querySelectorAll(".ldr-edit-btn").forEach(btn => {
  btn.addEventListener("click", () => editLeader(parseInt(btn.dataset.lid)));
});

// ❌ WRONG — causes issues with Arabic names containing special chars
h += `<button onclick="editLeader(${l.id})">تعديل</button>`;
```

### 2. ID type safety — always parseInt
```js
// Always parse id from dataset before using === comparison
const id = parseInt(btn.dataset.lid);
const leader = DATA.leaders.find(l => l.id === id); // === works after parseInt
```

### 3. Sheet writes — correct action + match field
```js
// Add new leader
sheetPost({ sheet: "القادة", action: "append", row: [name, cls, phone, dob] });

// Edit leader (match by name)
sheetPost({ sheet: "القادة", action: "update", match: oldName, row: [name, cls, phone, dob] });

// Delete leader
sheetPost({ sheet: "القادة", action: "delete", match: name });

// Add child to correct class tab
sheetPost({ sheet: ch.cls, action: "append", row: [name, phone, addr, dob, leaderName] });
```

### 4. Per-team leaders — never overwrite other team
```js
// When TL saves their plan, update only their team's key
setTeamLeaders(m.paras[k], CU.cls, selectedIds);
// This updates leaders_a OR leaders_b without touching the other
```

### 5. Date formatting
```js
// fDate() handles both ISO strings and plain YYYY-MM-DD
fDate("2026-10-01T21:00:00.000Z") // → "01 / 10 / 2026"
fDate("2026-10-01")               // → "01 / 10 / 2026"
```

---

## Pending / Known Issues

| # | Issue | Status |
|---|---|---|
| 1 | `mergeSheets()` not yet run — sheet still has old format | **⚠ Run mergeSheets() once** |
| 2 | `loadAll` supports both old and new format (auto-detects) | ✅ Done |
| 3 | PDF upload to Drive — requires `sheetPostWithResponse` (text/plain) | ✅ Done |
| 4 | Child attendance log (`حضور الأولاد`) not loaded back from sheet | 🔲 Not yet |
| 5 | Leader visitation log (`افتقاد القادة`) not loaded back | 🔲 Not yet |
| 6 | Offline PWA / Service Worker | 🔲 Not yet |

---

## How to Run Locally

```bash
# Clone
git clone https://github.com/nancybolis94-max/scout-app.git
cd scout-app

# Serve (needs a local server for fetch to work)
npx serve .
# OR
python3 -m http.server 8080
# Open http://localhost:8080
```

**Login credentials (from sheet):**
- Admin: password `1234`
- Team Leader ملايكة أ: password `1234`
- Team Leader ملايكة ب: password `1234`

---

## How to Deploy

```bash
git add .
git commit -m "feat: description"
git push origin main
# GitHub Pages auto-deploys from main branch
# Live in ~30 seconds at https://nancybolis94-max.github.io/scout-app/
```

---

## Apps Script Deployment Steps

1. Go to https://script.google.com
2. Open the project linked to the sheet
3. Paste content of `apps_script_v3.gs`
4. Click **Save**
5. Run `mergeSheets()` once: **Run > Run function > mergeSheets**
6. **Deploy > New deployment**
   - Type: Web app
   - Execute as: **Me**
   - Who has access: **Anyone**
7. Copy the new `/exec` URL
8. Update `AS_URL` constant in `index.html`
9. Push to GitHub

---

## Sheet Write Map (complete)

| Action | Sheet tab | row columns |
|---|---|---|
| Add leader | `القادة` | name, cls, phone, dob |
| Edit leader | `القادة` (update, match=name) | name, cls, phone, dob |
| Delete leader | `القادة` (delete, match=name) | — |
| Add child | `{child.cls}` | name, phone, addr, dob, leaderName |
| Edit child | `{child.cls}` (update, match=name) | name, phone, addr, dob, leaderName |
| Delete child | `{child.cls}` (delete, match=name) | — |
| Assign child leader | `توزيع الأولاد` (or col F after merge) | childName, leaderName, cls |
| TL plan save | `خطة الاجتماعات` (update, match=meetNum) | full 19-col row |
| Add meeting | `خطة الاجتماعات` | meetNum, date |
| Leader attendance | `حضور القادة` | date, name, status, uniform |
| Child attendance | `حضور الأولاد` | date, meetNum, childName, leaderName, status |
| Child followup | `الافتقاد` | date, leaderName, childName, status |
| Leader visitation | `افتقاد القادة` | date, cls, leaderName, status, notes |
| Add activity | `الأنشطة` | name, date, cls, leaderA, leaderB |
| Edit activity | `الأنشطة` (update, match=name) | name, date, cls, leaderA, leaderB |
| Add/edit password | `كلمات السر` | role, cls, password |
| Upload PDF | Drive action | base64, filename → returns viewUrl |
