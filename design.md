# Design: SocietyOS

Version 0.2 · Last updated 2026-10-25 · Companion to `prd.md`

**Direction:** brutalism that is functional and honest. Thick black borders, hard offset shadows with zero blur, visible structure, plain language, one brand colour (green) on neutral greys. It should feel like a well-made public notice board: confident, legible, no decoration for its own sake.

Colour intent from the founder brief: **light theme** = white and light greys; **dark theme** = greys shaded toward black; **green** is the shared accent in both; **black** is the line and text colour in light mode.

---

## 1. Design principles

1. **Clarity over beauty.** If a choice is a trade-off, pick the clearer one.
2. **Structure is visible.** Borders, grids, and labels show how things relate. No hidden affordances.
3. **One primary action per screen.** Always obvious, always green.
4. **Status is text first.** Never colour alone (see section 7).
5. **Mobile first.** Most residents use a phone. Design at 360 px, then expand.
6. **Same components everywhere.** Public site, resident app, and committee console share one kit; density differs, not style.
7. **Fast.** No heavy imagery, no animation dependency, no layout shift.

## 2. Colour tokens

Defined once as CSS variables in `packages/ui/src/tokens.css`. Components use tokens only; **no hex values in components**.

```css
:root {
  /* surfaces */
  --bg:         #F5F6F3;   /* page */
  --surface:    #E6E8E3;   /* cards, panels */
  --surface-2:  #D6D9D2;   /* nested, table header, hover */
  --ink:        #0A0A0A;   /* text and borders */
  --ink-muted:  #4A4F4A;   /* secondary text */
  --line:       #0A0A0A;   /* borders */
  --line-soft:  #B9BDB5;   /* dividers inside a card */

  /* brand green */
  --green-900:  #0F3D22;
  --green-700:  #1F6B3D;   /* primary action */
  --green-500:  #2E8B57;   /* accent, focus fill */
  --green-200:  #BFE3CB;   /* highlight, selected row */
  --primary:    var(--green-700);
  --primary-fg: #FFFFFF;

  /* feedback */
  --danger:     #B3261E;
  --warning:    #8A5F00;
  --success:    var(--green-700);
  --info:       #1E4E8C;

  --focus:      #0A0A0A;   /* 3px outline + 2px offset ring in bg colour */
  --shadow:     4px 4px 0 0 var(--ink);
  --shadow-sm:  2px 2px 0 0 var(--ink);
}

[data-theme='dark'] {
  --bg:         #0C0F0D;
  --surface:    #171B18;
  --surface-2:  #232824;
  --ink:        #ECEFEA;
  --ink-muted:  #A3AAA4;
  --line:       #ECEFEA;
  --line-soft:  #3A413C;

  --green-900:  #BFE3CB;
  --green-700:  #3FA566;   /* primary action */
  --green-500:  #55C080;
  --green-200:  #1F3A2A;
  --primary:    var(--green-700);
  --primary-fg: #0A0A0A;

  --danger:     #FF6B5E;
  --warning:    #E8B84A;
  --info:       #7FB0F0;
  --focus:      #ECEFEA;
}

@media (prefers-color-scheme: dark) {
  :root:not([data-theme='light']) { /* same values as [data-theme='dark'] */ }
}
```

Theme behaviour: default follows the OS; the toggle in the top bar stores the choice in a cookie (so the server renders the right theme without flash). Verified contrast: body text on `--bg` at least 15:1; `--primary-fg` on `--primary` at least 6:1; `--ink-muted` on `--bg` at least 7:1; warning and danger text on `--bg` at least 4.5:1 in both themes. Re-check any new token pair with axe before using it.

## 3. Typography

| Role | Font | Notes |
|---|---|---|
| UI and headings | Space Grotesk (fallback: Archivo, system-ui, sans-serif) | Loaded with `next/font`, `display: swap` |
| Data, IDs, chips, labels | JetBrains Mono (fallback: ui-monospace, monospace) | Complaint codes, amounts, dates in tables, status chips |

| Token | Size / line-height | Weight | Use |
|---|---|---|---|
| `display` | 48 / 52 | 700, tight tracking | Public hero only |
| `h1` | 32 / 38 | 700 | Page titles |
| `h2` | 24 / 30 | 700 | Section titles |
| `h3` | 18 / 24 | 700 | Card titles |
| `body` | 16 / 24 | 400 | Default text |
| `small` | 14 / 20 | 400 | Table cells, helper text |
| `label` | 12 / 16, mono, uppercase, +0.04em | 600 | Field labels, chips, table headers |

Rules: minimum body size 16 px on mobile; line length 60-80 characters for reading text; sentence case for headings (labels may be uppercase); numbers in tables use tabular figures (`font-variant-numeric: tabular-nums`) and are right-aligned.

## 4. Spacing, borders, shadows

- Spacing scale (px): 4, 8, 12, 16, 24, 32, 48, 64. Use the scale only.
- Border: **2px solid `--line`** on cards, inputs, buttons, tables, drawers. Dividers inside a card: 1px `--line-soft`.
- Radius: **0** everywhere (including inputs, buttons, chips, avatars). Map markers may be circular.
- Shadow: `--shadow` (4px 4px, no blur) on cards that are interactive or elevated and on buttons; `--shadow-sm` on chips and small controls. Pressed state: translate 2px 2px and shadow becomes `--shadow-sm`; no transition longer than 80 ms.
- Max content width: 1200 px for public pages, 1440 px for console; page gutters 16 px (mobile), 24 px (tablet), 32 px (desktop).
- Grid: 4 columns (mobile), 8 (tablet), 12 (desktop), 16 px gutter; the grid may be shown with 1px `--line-soft` rules on feature sections as a deliberate brutalist device.

## 5. Layout and navigation

Four layouts share the same shell:

| Layout | Used by | Structure |
|---|---|---|
| Public | Visitors | Top bar (logo, nav, theme toggle, Login), content, footer with committee contact |
| App (resident) | Residents | Top bar (search, bell, theme, profile) plus left rail on desktop; **bottom tab bar on mobile** (Home, Complaints, Book, Map, More) |
| Console (committee) | Committee, office bearers | Left rail with grouped sections, dense tables, filter bar, top bar as above |
| Admin | Admin | Same as console with a settings-oriented rail |

Persistent elements: "Raise complaint" primary action (top bar on desktop, floating square button on mobile), role switcher when the user has more than one role, society name and logo, theme toggle, notification bell with unread count.

Navigation labels (plain words): Home, Complaints, Book, Notices, Map, My property, Dues, Requests, History, Profile. Console adds: Board, Works, Bookings, Residents, Finance, Reports, Audit, Setup.

## 6. Components

Primitives live in `packages/ui` and are built on Radix UI headless components. Each has all states: default, hover, focus-visible, active, disabled, loading, error where relevant.

| Component | Spec |
|---|---|
| **Button** | Variants: `primary` (green fill, `--primary-fg` text), `secondary` (surface fill, ink text), `danger` (danger border and text, fill on hover), `link`. Sizes: sm 32 px, md 44 px (default, touch size), lg 52 px. 2px border, `--shadow`. Label in sentence case. Loading shows a square spinner and keeps width |
| **Input / Textarea / Select** | 44 px height, 2px border, `--bg` fill, label above (mono label style), helper text below, error text with an icon and `--danger`; focus: 3px outline `--focus` |
| **Checkbox / Radio / Switch** | Square controls with 2px border; checked = green fill with ink check |
| **Card** | 2px border, `--surface`, padding 16 or 24; interactive cards add `--shadow` and move on hover |
| **Chip (status/priority)** | Mono label, 2px border, 4px 8px padding, glyph prefix (section 7) |
| **Table** | Header row `--surface-2`, label style, sticky header, 1px row dividers, zebra optional, row hover `--surface-2`, selected `--green-200`; sortable columns show an arrow; wrapped in an `overflow-x: auto` container; on mobile, key tables switch to stacked cards |
| **Tabs** | Underline replaced by block tabs: active tab filled `--ink` with `--bg` text |
| **Dialog / Drawer** | 2px border, `--shadow`, overlay is 60% ink; drawer is the pattern for map details and quick views; focus trapped; Esc closes |
| **Toast** | Bottom-left (desktop) or top (mobile), 2px border, auto-dismiss 5 s except errors, always has text, optional action |
| **Tooltip** | Only for non-essential hints; never the only place for critical info |
| **Stat tile** | Large mono number, label, delta line, optional link to the filtered list |
| **Timeline** | Vertical 2px rule, square nodes, each entry: time, actor, action text, optional note; used for complaint events and service history |
| **Kanban** | Columns per status with counts; cards show code, title, category, priority chip, assignee initials, age; overdue cards get a danger left border |
| **Calendar** | Month, week, day; slots as blocks with text; blackout hatched; booked blocks show purpose (committee) or "Booked" (residents) |
| **Avatar** | Square, initials, 2px border |
| **Empty state** | Boxed message, one sentence of explanation, one primary action |
| **Skeleton** | Flat `--surface-2` blocks with a stepped 2-frame blink (no gradients); disabled when `prefers-reduced-motion` |
| **Pagination / Load more** | Prev/Next buttons with "Showing 1-25"; infinite scroll is not used for tables |
| **Breadcrumb** | Mono, slash-separated, on console pages |
| **Banner** | Full-width strip for system messages (maintenance, offline), 2px bottom border |

## 7. Status, priority, and feedback language

Status is never conveyed by colour alone: every chip has **text and a glyph**.

| Complaint status | Chip label | Glyph | Style |
|---|---|---|---|
| submitted | SUBMITTED | ○ | outline |
| acknowledged | ACKNOWLEDGED | ◔ | `--surface-2` fill |
| assigned | ASSIGNED | ◑ | `--green-200` fill |
| in_progress | IN PROGRESS | ◕ | `--green-500` fill, ink text |
| on_hold | ON HOLD | ‖ | dashed border, warning text |
| resolved | RESOLVED | ✓ | `--green-700` fill, `--primary-fg` text |
| closed | CLOSED | ■ | `--ink` fill, `--bg` text |
| rejected | REJECTED | ✕ | danger border and text |
| reopened | REOPENED | ↺ | danger fill, `--bg` text |
| overdue (flag) | OVERDUE | ! | danger fill, appears next to the status chip |

| Priority | Chip | Style |
|---|---|---|
| critical | CRITICAL ▲▲▲ | danger fill |
| high | HIGH ▲▲ | danger outline |
| normal | NORMAL ▲ | outline |
| low | LOW ▽ | `--surface-2` fill |

Use the same pattern for booking, invoice, and project statuses: solid green = good/complete, outline = pending, dashed = waiting on someone else, danger = failed or overdue.

## 8. Page wireframes

Text wireframes define structure and priority, not final styling.

### 8.1 Public home

```
+--------------------------------------------------------------+
| [LOGO] Society Name        About  Amenities  Notices  Events |
|                              Contact          [Theme] [LOGIN]|
+--------------------------------------------------------------+
| HERO (display type)                                          |
| One sentence about the society.            [ Resident login ]|
+----------------------+----------------------+----------------+
| PUBLIC NOTICES       | UPCOMING EVENTS      | AMENITIES      |
| ▪ title  date        | ▪ title  date        | [img] Hall     |
| ▪ title  date        | ▪ title  date        | [img] Ground   |
+----------------------+----------------------+----------------+
| SOCIETY MAP (public layer: zones + amenities)  [ Open map ]  |
+--------------------------------------------------------------+
| Committee contacts · Enquiry form · Footer                   |
+--------------------------------------------------------------+
```

### 8.2 Resident dashboard

```
+--------------------------------------------------------------+
| Hello, <Name> · Property D2-14                [ + Complaint ]|
+---------------+---------------+---------------+--------------+
| OPEN          | BOOKINGS      | DUES          | NOTICES      |
| COMPLAINTS 2  | NEXT: Sat 6pm | ₹ 2,400 due   | 3 unread     |
+---------------+---------------+---------------+--------------+
| MY COMPLAINTS (latest 5, with status chips)  | EVENTS        |
| CMP-2026-000123 Leak in bathroom  IN PROGRESS| ▪ AGM  Sun    |
| ...                                          | ▪ Holi  14 Mar|
+----------------------------------------------+---------------+
| PINNED NOTICES                                               |
+--------------------------------------------------------------+
```

### 8.3 Raise complaint (mobile-first, single column)

```
CATEGORY   [ Plumbing        v ]
TITLE      [__________________ ]
DETAILS    [__________________ ]
LOCATION   ( ) My property  ( ) Common area  ( ) Pick on map
PHOTOS     [ + add ]  [img][img]
PRIORITY   [ Normal v ]   VISIBILITY ( ) Only me ( ) Everyone
[ Similar open complaints (3) ]   <- duplicate hint
[ SUBMIT COMPLAINT ]
```

### 8.4 Complaint detail

```
CMP-2026-000123            [ IN PROGRESS ] [ HIGH ▲▲ ] [ OVERDUE ]
Title · Category · Location (link to map) · Raised by · Date
Assigned to: <Name> (Water & Electrical)   Due: <date/time>
-------------------------------------------------------------
Description + photos
-------------------------------------------------------------
TIMELINE (events)        |  COMMENTS (public / internal tabs)
-------------------------------------------------------------
Actions (role-based): Assign · Change status · Resolve · Reopen · Confirm
```

### 8.5 Committee board

```
FILTERS: Department v  Zone v  Priority v  Age v  Assignee v  [Overdue only]
+ SUBMITTED (4) + ASSIGNED (6) + IN PROGRESS (9) + ON HOLD (2) + RESOLVED (5)
| card | card | card | ... horizontally scrollable, drag to change status
Toggle: [Board] [Table]            Export CSV
```

### 8.6 Map page

```
+----------------------+-----------------------------------------+
| LAYERS (legend)      |                                         |
| [x] Zones            |           MAP CANVAS                    |
| [x] Properties       |   D1 | D2 | D3 | D4                      |
| [ ] Water   ——       |   D5 | D6 | ...                         |
| [ ] Drainage - -     |                                         |
| [ ] Electrical ·-·   |           [+][-] [List view]            |
| [ ] Works ////       |                                         |
+----------------------+-----------------------------------------+
| DETAIL DRAWER (on click): zone totals or property details +     |
| history timeline + [Raise complaint here]                        |
+-----------------------------------------------------------------+
```
Mobile: layers in a bottom sheet, detail drawer slides up from the bottom, "List view" always one tap away.

### 8.7 Booking

```
RESOURCE: [ Community Hall v ]  Rules: 9am-10pm · max 4h · ₹1,500 · approval needed
<  October 2026  >   [Day][Week][Month]
calendar grid with booked / blocked / available slots
[ Request this slot ] -> purpose, guests, summary of fee, [ SEND REQUEST ]
```

## 9. States

| State | Pattern |
|---|---|
| Loading | Skeleton blocks matching the final layout; never spinners over blank pages for more than 300 ms |
| Empty | Boxed message: what this area is, why it is empty, one action ("No complaints yet. Raise your first complaint.") |
| Error (recoverable) | Inline banner with plain explanation and a Retry button; keep user input |
| Error (not found) | "We could not find that. It may have been removed or you may not have access." with a link home. Do not reveal existence |
| Forbidden action | Disabled control with a short reason where useful, otherwise hidden; server still enforces |
| Offline | Top banner "You are offline. Changes will not be saved." |
| Success | Toast plus the updated UI; destructive success (cancel, reject) states what changed |

## 10. Map styling

Chrome stays green and grey; **utility lines use a dedicated data palette** so each parameter is distinguishable (as in the founder sketch: different colour, different parameter). Every line also has a dash pattern so the map is readable in grayscale and for colour-blind users.

| Layer | Light | Dark | Dash / style | Width |
|---|---|---|---|---|
| Water | `#1E6FD9` | `#5AA2FF` | solid | 3 |
| Drainage | `#7A4E2D` | `#C79A73` | dashed (8 4) | 3 |
| Electrical | `#B58900` | `#F2C94C` | dash-dot (10 3 2 3) | 3 |
| Gas | `#D9480F` | `#FF8A5B` | dotted (2 4) | 3 |
| Internet | `#7B3FC4` | `#B590F0` | long dash (14 6) | 2 |
| Fire | `#B3261E` | `#FF6B5E` | solid, double width | 4 |

| Feature | Style |
|---|---|
| Zone | 2px `--line` outline, `--surface` fill at 60%, zone code label in mono at centre; selected: `--green-200` fill and 3px outline |
| Property | 1px outline, `--bg` fill; own property: `--green-500` fill; committee mode can colour by complaint count (stepped scale with legend and numbers) |
| Amenity | Square marker with an icon and text label on hover and in the list |
| Active works | Diagonal hatch pattern over affected area, `--warning` outline |
| Asset | Small square marker; type icon; selected has a thick ring |
| Complaint pins | Squares with count; clustered when zoomed out |

Legend is always visible and doubles as the layer toggle. Labels never rely on colour alone. The base map is blank `--bg` with a light grid; optional OpenStreetMap or satellite backdrop is toggled by admin.

## 11. Forms and validation

- One column; labels above fields; required fields marked with "(required)" text, not just an asterisk.
- Validate on blur and on submit; show the first error summary at the top with links to fields, and inline messages.
- Never clear user input on error. Preserve drafts of long text locally (per form) for complaints and notices.
- Buttons describe the outcome: "Submit complaint", "Send request", "Approve booking", not "Submit" or "OK".
- Destructive actions use a confirmation dialog stating what will happen and require a reason where the PRD says so.
- Dates and times are shown in the society timezone with the timezone label on booking and invoice screens. Currency as `₹ 1,23,456` (Indian grouping) via `Intl.NumberFormat('en-IN')`.

## 12. Motion

Minimal. Allowed: 80 ms press feedback, 120 ms drawer slide, instant theme switch. No parallax, no scroll-jacking, no looping animation. Respect `prefers-reduced-motion` by removing all transitions and the skeleton blink.

## 13. Accessibility (WCAG 2.2 AA)

- Visible focus on every interactive element (3px outline, never removed).
- Full keyboard operation including the map (arrow keys pan, +/- zoom, Enter opens the selected feature) and the Kanban board (move card via menu, drag is optional).
- Hit targets at least 44 x 44 px on touch.
- Semantic landmarks (`header`, `nav`, `main`, `aside`, `footer`), one `h1` per page, heading order respected.
- Every icon-only button has an accessible name; images have alt text; decorative images are empty-alt.
- Colour is never the only carrier of meaning (chips, map lines, charts).
- Form errors announced via `aria-live`; toasts use `role="status"` (errors `role="alert"`).
- Tables have `th` scopes and captions; stacked mobile cards keep label-value pairing.
- axe runs in CI on key pages in both themes; a failing check blocks merge.

## 14. Responsive behaviour

| Breakpoint | Behaviour |
|---|---|
| 360-639 | Single column, bottom tab bar, drawers become bottom sheets, tables become cards |
| 640-1023 | Two columns where useful, rail collapses to icons |
| 1024+ | Full rail, multi-column dashboards, side drawers |
| 1440+ | Console content max width, extra whitespace rather than stretched tables |

Test every key flow at 360 x 640, 768 x 1024, and 1440 x 900.

## 15. Content and microcopy

- Plain, direct, respectful. Short sentences. No jargon, no exclamation marks, no emojis.
- Say what happened and what to do next: "Complaint submitted. We have sent it to Water and Electrical. You will be notified when it is assigned."
- Use the same nouns everywhere: **complaint** (not ticket or issue), **resident** (not user), **committee**, **property** (not unit, except in data fields), **booking**, **notice**.
- All UI strings come from message files keyed by feature (`complaints.raise.submit`), ready for Hindi later.
- Error messages never blame the user and never show technical details.

## 16. Iconography and imagery

- Icon set: Lucide, 2px stroke, square caps where possible, 20 px default. Icons support text; they do not replace it.
- Photography: real photos of the society, black-and-white or low-saturation treatment with a 2px border; no stock people.
- Logo: wordmark in Space Grotesk bold on a green square; final mark pending the product name.

## 17. Don'ts

- No gradients, blur, glassmorphism, or soft shadows.
- No rounded corners.
- No colour-only status.
- No hex values or one-off spacing in components.
- No new component when an existing one can be extended; add variants to `packages/ui`.
- No auto-playing media, carousels for critical content, or modal-on-load.
